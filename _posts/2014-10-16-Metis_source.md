---
layout: post
title:	Metis源码分析
category:	MapReduce, Distributed System
---
## Metis源码分析
&emsp;前些天花了些时间把MIT的分布式课程6.824做完了， git里显示这门课的助教叫Yandong Mao，看样子是个中国人。于是到了MIT PDOS的主页上查了查，找到了这个人的github。没想到在他的github上发现了一个更有趣的东西：MapReduce的C++实现。而且还有一篇文章Optimizing MapReduce for Multicore Architectures.讲他在这里面主要做的优化，于是把文章下载下来读了一遍，准备再开始把这个开源实现的代码读一下。

&emsp;由于源码不好找一个明显的入口，于是便想采用case-study的方式，从里面的wc例程开始追踪，一步步了解MapReduce的运行机制。

&emsp;wc例程的main函数中，在解析完命令行参数之后，有如下代码段：

    .....
    mapreduce_appbase::initialize();
    /*get input file */
    wc app(fn, map_tasks);
    app.set_ncore(nprocs);
    ....

看样子是先将mapreduce_appbase这个类初始化。在lib/application.cc里面可以找到这个函数的实现：
    
    void mapreduce appbase::initialize() {
        threadinfo::initialize();
    }

可见个这个初始化函数做的事情很简单，只是调用了threadinfo的初始化接口，估计是新创建了一些工作线程，于是再跳到threadinfo::initialize()的代码中：

    static void initalize() {
		assert(pthread_key_create(&key_, free) == 0);
		created_ = true;
	}

果然，先执行了pthread\_key\_create()函数，然后将created_标志位设成true

再回到wc.cc的main函数中，接下来执行的是wc app(fn, map_tasks)语句，于是，查看struct wc的构造函数：

	struct wc : public map_reduce {
		wc(const char *f, int nsplit) : s_(f, nsplit) {}
	....
s_是struct wc的一个private成员：

	....
	private:
	defsplitter s_;
	...
我们再来看struct defsplitter的构造函数：

	struct defsplitter {
    	defsplitter(char *d, size_t size, size_t nsplit)
       	 : d_(d), size_(size), nsplit_(nsplit), pos_(0) {
        pthread_mutex_init(&mu_, 0);
   	 	}
    	defsplitter(const char *f, size_t nsplit) : nsplit_(nsplit), pos_(0), mf_(f) {
        	pthread_mutex_init(&mu_, 0);
        	size_ = mf_.size_;
        	d_ = mf_.d_;
    	}
	...
显然这里用的是第二个构造函数，在初始化列表时，调用mf_的构造函数将mf_初始化，mf_为struct defsplitter的一个private成员：

	struct defsplitter {
		......
		private:
    	char *d_;
    	size_t size_;
    	size_t nsplit_;
    	size_t pos_;
    	mmap_file mf_;
    	pthread_mutex_t mu_;
	};

再来看mmapfile类的定义：

	struct mmap_file {
    mmap_file(const char *f) {
        assert((fd_ = open(f, O_RDONLY)) >= 0);
        struct stat fst;
        assert(fstat(fd_, &fst) == 0);
        size_ = fst.st_size;
        d_ = (char *)mmap(0, size_ + 1, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd_, 0);
        assert(d_);
    }
    mmap_file() : fd_(-1) {}
    virtual ~mmap_file() {
        if (fd_ >= 0) {
            assert(munmap(d_, size_ + 1) == 0);
            assert(close(fd_) == 0);
        }
    }
    char &operator[](off_t i) {
        return d_[i];
    }
    size_t size_;
    char *d_;
	private:
	int fd_;
	};	

    
不难看出，设置这个类的目的，便是要将整个文件的内容mmap到内里里来，方便以后的访问。代码很容易看懂，不多作分析。

继续回到wc.cc的main函数中，接下来执行的两句是：

	wc app(fn, map_tasks);
    app.set_ncore(nprocs);
    app.set_reduce_task(reduce_tasks);
	app.sched_run();

很容易理解，先是设置了使用的核数，然后设置了reduce_task的数目，

但是要注意的是，**set\_ncore函数是继承于mapreduce_appbase类的，而set_reduce_task是继承于map_reduce类的。**，至于为什么要这样承继，我的看法是为了解耦，让最底层的类负责根据所用的CPU数创建所需要的线程，让只与map_reduce直接相关的类负责决定使用多少个map(即将原始数据分成多少片，以及多少个reduce。

然后执行app.sched\_run\(\)，这是我们要重点分析的地方。 MapReduce的主要过程也基本上都集中在了这里（根据我们刚才的推测，由于只是确定了所用的线程数但是还没有真正的创建线程，所以sched\_run应该先到mapreduce\_appbase类里面创建所需要的线程，再到map_reduce里面给线这些线程分配任务)：

    int mapreduce_appbase::sched_run() {
        assert(threadinfo::initialized() && "Call mapreduce_apppase::initialize first");
        static_appbase::set_app(this);
        assert(clean_);
        clean_ = false;
        const int max_ncore = get_core_count();//得到当前机器上可用的最大CPU个数
        assert(ncore_ <= max_ncore);
        if (!ncore_)
    	ncore_ = max_ncore;
    
        verify_before_run();
        // initialize threads
        mthread_init(ncore_);//根据之前指定的核数，创建所需要的线程
    
        // pre-split
        ma_.clear();//ma_为mpareduce_appbase类的private成员
        split_t ma;
        bzero(&ma, sizeof(ma));
        while (split(&ma, ncore_)) {  //根据所用手线程数，将数据进行分片
            ma_.push_back(ma);
            bzero(&ma, sizeof(ma));
        }
        uint64_t real_start = read_tsc();
        // get the number of reduce tasks by sampling if needed
        if (skip_reduce_or_group_phase()) {
            m_ = create_map_bucket_manager(ncore_, 1);
            get_reduce_bucket_manager()->init(ncore_);
        } else {
    	if (!nreduce_or_group_task_)//如果没有在参数里设定reduce的数量，则先进行采样，以决定reduce的数量
    	    nreduce_or_group_task_ = sched_sample();
            m_ = create_map_bucket_manager(ncore_, nreduce_or_group_task_);//根据map, reduce的数目创建map阶段所需要的hash bucket
            get_reduce_bucket_manager()->init(nreduce_or_group_task_);
        }
    
        uint64_t map_time = 0, reduce_time = 0, merge_time = 0;
        // map phase
        run_phase(MAP, ncore_, map_time, nsample_);
        // reduce phase
        if (!skip_reduce_or_group_phase())
    	run_phase(REDUCE, ncore_, reduce_time);
        // merge phase
        const int use_psrs = USE_PSRS;
        if (use_psrs) {
            merge_ncore_ = ncore_;
    	run_phase(MERGE, merge_ncore_, merge_time);
        } else {
            reduce_bucket_manager_base *r = get_reduce_bucket_manager();
    	merge_ncore_ = std::min(int(r->size()) / 2, ncore_);
    	while (r->size() > 1) {
    	    run_phase(MERGE, merge_ncore_, merge_time);
                r->trim(merge_ncore_);
    	    merge_ncore_ /= 2;
    	}
        }
        set_final_result();
        total_map_time_ += map_time;
        total_reduce_time_ += reduce_time;
        total_merge_time_ += merge_time;
        total_real_time_ += read_tsc() - real_start;
        reset();  // result everything except for results_
        return 0;
    }

mthread_init的代码：

    void mthread_init(int ncore) {
        if (tp_created_)
            return;
        threadinfo *ti = threadinfo::current();
        cpumap_init();
        ncore_ = ncore;
        ti->cur_core_ = main_core;
        assert(affinity_set(cpumap_physical_cpuid(main_core)) == 0);
        tp_created_ = true;
        bzero(tp_, sizeof(tp_));
        for (int i = 0; i < ncore_; ++i)
    	if (i == main_core)
    	    tp_[i].tid_ = pthread_self();
    	else
    	    assert(pthread_create(&tp_[i].tid_, NULL, mthread_entry, int2ptr(i)) == 0);
    }
先得到指向当前threadinfo的指针，然后将当前ti\-\>cur\_core设置为main\_core(这个值是一个枚举值，为0)。将当前线程绑定到main_core上面，然后，根据所需要的线程数，一个一个的创建新线程，新线程执行的函数为mthread、_entry，参数为线程所在的CPU编号。

    void *mthread_entry(void *args) {
        threadinfo *ti = threadinfo::current();
        ti->cur_core_ = ptr2int<int>(args);
        assert(affinity_set(cpumap_physical_cpuid(ti->cur_core_)) == 0);//将新线程也绑定
        while (true)
            tp_[ti->cur_core_].run_next_task();//死循环，永远执行athread_type里的run_next_task函数
    }
athread_type类如下：

    struct  __attribute__ ((aligned(JOS_CLINE))) athread_type {
        void *volatile a_;
        void *(*volatile f_) (void *);
        volatile char pending_;
        pthread_t tid_;
        volatile bool running_;
    
        template <typename T>
        void set_task(void *arg, T &f) {
            a_ = arg;
            f_ = f;
            mfence();
            pending_ = true;
        }
    
        void wait_finish() {
            while (running_)
                nop_pause();
        }
    
        void wait_running() {
            while (pending_)
                nop_pause();
        }
        void run_next_task() {
            while (!pending_)
                nop_pause();
            running_ = true;
            pending_ = false;
            f_(a_);
            running_ = false;
        }
    };

由于athread_type tp_[JOS_NCPU]为一个未初始化的全局变量，而未初始化的全局变量默认是会清零的，所以，开始运行时，pending_为0，于是一直执行nop_pause()函数，让CPU什么也不做，等待map任务的来临。

	struct mapreduce_appbase {
		....
        int next_task_;
        int phase_;
        xarray<split_t> ma_;
		....
xarray为一个模版类，split_t的类型定义如下：

    struct split_t {
        void *data;
        size_t length;
    };
数据分片完成以后，开始创建map阶段所需要的hash bucket，也就是构建那个矩阵。

    map_bucket_manager_base *mapreduce_appbase::create_map_bucket_manager(int nrow, int ncol) {
        enum { index_append, index_btree, index_array };
        int index = (application_type() == atype_maponly) ? index_append : DEFAULT_MAP_DS;
        map_bucket_manager_base *m = NULL;
        switch (index) {
        case index_append:
    #ifdef SINGLE_APPEND_GROUP_FIRST
            m = new map_bucket_manager<false, keyval_arr_t, keyvals_t>;
    #else
            m = new map_bucket_manager<false, keyval_arr_t, keyval_t>;
    #endif
            break;
        case index_btree:
            typedef btree_param<keyvals_t, static_appbase::key_comparator, 
                                static_appbase::key_copy_type, static_appbase::value_apply_type> btree_param_type;
            m = new map_bucket_manager<true, btree_type<btree_param_type>, keyvals_t>;
            break;
        case index_array://对于wc例程，默认使用的是这种方式
            m = new map_bucket_manager<true, keyvals_arr_t, keyvals_t>;
            break;
        default:
            assert(0);
        }
        m->global_init(nrow, ncol);//创建完合适的实例后，统一使用global_init函数进行初始化。
        return m;
    };

map\_bucket\_manager是一个模板类，其定义如下：

 	template <bool S, typename DT, typename OPT>
    struct map_bucket_manager : public map_bucket_manager_base {
	.....
      private:
        DT *mapdt_bucket(size_t row, size_t col) {
            return mapdt_[row]->at(col);
        }
        ~map_bucket_manager() {
            reset();
        }
        psrs<C> pi_;
        size_t rows_;
        size_t cols_;
        xarray<xarray<DT> *> mapdt_;  // intermediate ds holding key/value pairs at map phase
        xarray<C> output_;
    };

而m的类型为map\_bucket\_manager\_base *，可知这里是用了基类的指针来指向子类，来使用子类中多态的函数。

注意map_bucket_manager里使用的参数为:map\_bucket\_manager<true, keyvals\_arr\_t, keyvals\_t>。

来看global\_init函数：

    template <bool S, typename DT, typename OPT>
    void map_bucket_manager<S, DT, OPT>::global_init(size_t rows, size_t cols) {
        mapdt_.resize(rows);
        output_.resize(rows * cols);
        for (size_t i = 0; i < output_.size(); ++i)
            output_[i].init();
        rows_ = rows;
        cols_ = cols;
    }
这里主要初始的是map_bucket_manager类里的mapdt_成员和output_成员，这两个成员在struct map+_bucket_manager里的定义如下：

        xarray<xarray<DT> *> mapdt_;  // intermediate ds holding key/value pairs at map phase
        xarray<C> output_;

结合上面的参数map\_bucket\_manager<true, keyvals\_arr\_t, keyvals\_t>，我们可以知道，mapdt_真正被实例化的类型为：

	xarray<xarray<keyvals_arr_t> mapdt_;

而且mapdt\_的size被设置成为了rows，那么，我们可以知道，

	xarray<keyvals_arr_t>

为其map产生的结果中，存储每一行所使用的数据结构，同理，

	keyvals_arr_t

为其存储每一个hash bucket所使用的数据结构，我们把它称为cell。
那么，每一个cell又是怎么存放的呢？  幅理论上可以矮星，每一个cell中，应该存放的是hash值在同一个聚会范围内的，若干key-value pairs构成的集合，而对于一个特定的key-value pair来说，其key值唯一，但是value值可能有多个（因为同一个key在同一个map任务中可能会出现多次)。下面是这个数据结构的定义：


    struct keyvals_arr_t : public xarray<keyvals_t> {
        bool map_insert_sorted_copy_on_new(void *k, void *v, size_t keylen, unsigned hash);
        void map_insert_sorted_new_and_raw(keyvals_t *p);
    };

可以看到，keyvals\_arr\_t本质上还是一个xarray<keyvals\_t>，其中xarray中的每一个元素对应着一个具体的hash值， 而对于一个具体的hash值来说，存放其key-value的结构为keyvals\_t，其定义如下：


    struct keyvals_t : public xarray<void *> {
        void *key_;			/* put key at the same offset with keyval_t */
        unsigned hash;
        keyvals_t() {
            init();
        }
		...
	};

呵呵，居然本质上还是一个xarray<void *>，不过这已经是取后一环了，void *key_和unisgned hash变量存储了一个具体的key的信息，而对于这个key可能出现很多次的情况，则将这个key对应的每一个值都放到xarray<void *>里面。

这样，对于map阶段使用的数据结构就比较清晰了。这样才可以理解后面的map执行的过程。

在sched_run函数里，初始化完map阶段所需要的数据结构后，便开始运行map阶段。

    int mapreduce_appbase::sched_run() {
	...
    	if (!nreduce_or_group_task_)
    	    nreduce_or_group_task_ = sched_sample();
            m_ = create_map_bucket_manager(ncore_, nreduce_or_group_task_);
            get_reduce_bucket_manager()->init(nreduce_or_group_task_);
        }
    
        uint64_t map_time = 0, reduce_time = 0, merge_time = 0;
        // map phase
        run_phase(MAP, ncore_, map_time, nsample_);
        // reduce phase
        if (!skip_reduce_or_group_phase())
    	run_phase(REDUCE, ncore_, reduce_time);
	...

可以看到，map阶段和reduce阶段有一个统一的入口:run\_phase：

    void mapreduce_appbase::run_phase(int phase, int ncore, uint64_t &t, int first_task) {
        uint64_t t0 = read_tsc();
        prof_phase_init();
        pthread_t tid[JOS_NCPU];
        phase_ = phase;
        next_task_ = first_task;
        for (int i = 0; i < ncore; ++i) {
    	if (i == main_core)
    	    continue;
    	mthread_create(&tid[i], i, base_worker, this);
        }
        mthread_create(&tid[main_core], main_core, base_worker, this);
        for (int i = 0; i < ncore; ++i) {
    	if (i == main_core)
    	    continue;
    	void *ret;
    	mthread_join(tid[i], i, &ret);
        }
        prof_phase_end();
        t += read_tsc() - t0;
    }

先跳过main_core上的线程，对于nocre中其它的每个线程，执行mthread_create函数，传进去的函数指针为base_worker：


    void mthread_create(pthread_t * tid, int lid, void *(*start_routine) (void *),
      	            void *arg) {
        assert(tp_created_);
        if (lid == main_core)
    	start_routine(arg);
        else {
            tp_[lid].wait_finish();
    	tp_[lid].set_task(arg, start_routine);
            tp_[lid].wait_running();
        }
    }

如果是main\_core的话，则直接执行start\_routine(这里即为base\_worker函数，如果不是main\_core，那么先wait_finish()，然后将start\_routine这个函数指针挂上去，然后开始执行，具体很好理解，这里不再下到各个函数里面进行分析，而转过来看传进来的函数指针base\_worker:


    void *mapreduce_appbase::base_worker(void *x) {
        mapreduce_appbase *app = (mapreduce_appbase *)x;
        threadinfo *ti = threadinfo::current();
        prof_worker_start(app->phase_, ti->cur_core_);
        int n = 0;
        const char *name = NULL;
        switch (app->phase_) {
        case MAP:
            n = app->map_worker();
            name = "map";
            break;
        case REDUCE:
            n = app->reduce_worker();
            name = "reduce";
            break;
        case MERGE:
            n = app->merge_worker();
            name = "merge";
            break;
        default:
            assert(0);
        }
        dprintf("total %d %s tasks executed in thread %ld(%d)\n",
    	    n, name, pthread_self(), ti->cur_core_);
        prof_worker_end(app->phase_, ti->cur_core_);
        return 0;
    }

可见，再一次采用了代码利用机制，这里由于执行的map phase，那么便开始执行map_worker()函数：

    int mapreduce_appbase::map_worker() {
        threadinfo *ti = threadinfo::current();
        (sampling_ ? sample_ : m_)->per_worker_init(ti->cur_core_);
        if (!sampling_ && sample_)  //如果现在非采样中，并且之前有过采样，那么便将之前采样过的key-value pairs转移到新的hash bucket中来，在后面分析采样原理时再来分析
            m_->rehash(ti->cur_core_, sample_);
        int n, next;
        for (n = 0; (next = next_task()) < int(ma_.size()); ++n) { //ma_.size()为之前数据分片的片数，next_task()返回的是一个所有线程公用的变量，即只要全局中还有map任务没有完成，该线程便继续执行。
    	map_function(ma_.at(next));//最核心的map函数。将需要处理的数据的分片地址传进去
            if (sampling_)
    	    e_[ti->cur_core_].task_finished();
        }
        if (!sampling_ && skip_reduce_or_group_phase()) {
            m_->prepare_merge(ti->cur_core_);
            if (application_type() == atype_maponly) {
    #ifndef SINGLE_APPEND_GROUP_FIRST
                typedef map_bucket_manager<false, keyval_arr_t, keyval_t> expected_mtype;
                expected_mtype* m = static_cast<expected_mtype*>(m_);
                auto output = m->get_output(ti->cur_core_);
    
                reduce_bucket_manager_base *rb = get_reduce_bucket_manager();
                typedef reduce_bucket_manager<keyval_t> expected_rtype;
                expected_rtype *x = static_cast<expected_rtype *>(rb);
                assert(x);
                x->set(ti->cur_core_, output);
    #endif
            }
        }
        return n;
    }

map\_function是用户定义的，在wc例程序里定义如下：

    struct wc : public map_reduce {
        wc(const char *f, int nsplit) : s_(f, nsplit) {}
        bool split(split_t *ma, int ncores) {
            return s_.split(ma, ncores, " \t\r\n\0");
        }
        int key_compare(const void *s1, const void *s2) {
            return strcmp((const char *) s1, (const char *) s2);
        }
        void map_function(split_t *ma) {
            char k[1024];
            size_t klen;
            split_word sw(ma);
            while (sw.fill(k, sizeof(k), klen))//获取一个单调，首地址指针为k，单词长度为klen
                map_emit(k, (void *)1, klen);//将这个单词插入到map的矩阵中,相当于key = 这个单词，value = 1 
        }
		...
	}

mep_emit函数如下：

    void mapreduce_appbase::map_emit(void *k, void *v, int keylen) {
        unsigned hash = partition(k, keylen);//先通过hash函数计算出当前key所对应的hash值
        threadinfo *ti = threadinfo::current();
        bool newkey = (sampling_ ? sample_ : m_)->emit(ti->cur_core_, k, v, keylen, hash);//由于当前不是处于采样阶段，所以执行的是m_->emit()函数，m_即为刚才初始化map阶段所需数据结构时生成的map阶段的矩阵
        if (sampling_)
            e_[ti->cur_core_].onepair(newkey);
    }

emit将执行下面的函数：

    template <bool S, typename DT, typename OPT> // S = true, DT = keyvals_arr_t ODT = keyvals_t
    bool map_bucket_manager<S, DT, OPT>::emit(size_t row, void *k, void *v,
                                              size_t keylen, unsigned hash) {
        DT *dst = mapdt_bucket(row, hash % cols_); //需要插入到的cell
        return map_insert_analyzer<DT, S>::copy_on_new(dst, k, v, keylen, hash);
    }
由于S = true，接下来将执行以下函数：

    template <typename DT>
    struct map_insert_analyzer<DT, true> {
        static bool copy_on_new(DT *dst, void *key, void *val, size_t keylen, unsigned hash) {
            return dst->map_insert_sorted_copy_on_new(key, val, keylen, hash);
        }
        typedef typename DT::element_type T;
        static void insert_new_and_raw(DT *dst, T *t) {
            dst->map_insert_sorted_new_and_raw(t);
        }
    };

然后进入：

    bool keyvals_arr_t::map_insert_sorted_copy_on_new(void *key, void *val, size_t keylen, unsigned hash) {
        keyvals_t tmp(key, hash);
        size_t pos = 0;
        bool newkey = atomic_insert(&tmp, static_appbase::pair_comp<keyvals_t>, &pos);
        if (newkey)//如果发现当前key值没有在目标cell中出现，则将这个hash值对应的key复制到hash bucket中
            at(pos)->key_ = static_appbase::key_copy(key, keylen);//本质是memcpy，将数据分片中的key复制到hash bucket中，不多作分析
        at(pos)->map_value_insert(val);//将key所对应的value更新到hash bucket，需要分析一下
        return newkey;
    }

由于初始化时，默认使用的存储一个hash bucket中所有key-value paires的方式是arraY,则接下来会进入到以下的函数：

        template <typename F>
        bool atomic_insert(const T *e, const F &cmp, size_t *ppos = NULL) {
            bool bfound = false;
            size_t pos = lower_bound(e, cmp, &bfound);//在目标hash桶中寻找插入e时，e应处的位置，用的是二分查找，不再深入分析
            if (ppos)
                *ppos = pos;//记录位置
            if (bfound)//如果发现这个hash值已经存在，那么返回false
                return false;
            insert(pos, e);//如果该hash值不存在，将其插入当前cell，并返回true，但是注意，这里只是插入了一个空的临时的key,其正式的key和value都还没有插入进来
            return true;
        }

map_value_insert函数如下：

    void keyvals_t::map_value_insert(void *v) {
        static_appbase::map_values_insert(this, v);
    }
    
实际上是：

        static void map_values_insert(keyvals_t *dst, void *v) {
            return the_app_->map_values_insert(dst, v);
        }

即：

    void map_reduce::map_values_insert(keyvals_t *kvs, void *v) {
        if (has_value_modifier()) {//对于wc来说，这个值为真
            if (kvs->size() == 0)//如果这个 key是第一次出现，直接将值设定即可
                kvs->set_multiplex_value(v);
            else
    	    kvs->set_multiplex_value(modify_function(kvs->multiplex_value(), v));// 先执行modify_function，然后执行set_multiplex_value
    	return;
        }
        kvs->push_back(v);
        if (kvs->size() >= combiner_threshold) {
    	size_t newn = combine_function(kvs->key_, kvs->array(), kvs->size());
            assert(newn <= kvs->size());
            kvs->trim(newn);
        }
    }

对于wc例程，modify_function如下：

        void *modify_function(void *oldv, void *newv) {
            uint64_t v = (uint64_t) oldv;
            uint64_t nv = (uint64_t) newv;
            return (void *) (v + nv);
        }

呵呵，其实就是把两个值相加，这样可以减少reduce阶段的压力。


至此，map阶段的流程大体上就分析完了，后面开始分析reduce阶段。