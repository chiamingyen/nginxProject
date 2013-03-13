handler模組(100%)
========================

handler模組簡介
-----------------------

相信大家在看了前一章的模組概述以後，都對nginx的模組有了一個基本的認識。基本上作為第三方開發者最可能開發的就是三種類型的模組，即handler，filter和load-balancer。Handler模組就是接受來自用戶端的請求並產生輸出的模組。至於有些地方說的upstream模組則實際上也是一種handler。只不過它產生的內容來自於從後端伺服器獲取的，而非在本機產生的。

當Nginx系統啟動的時候，每個handler都有一次機會把自己關聯到一個在配置檔中使用location指令配置的一個location上。如果有多個handler模組都去關聯同一個location，那麼實際上只有一個handler模組真正會起作用。當然大多數情況下，模組開發人員都會避免出現這種情況。

一個handler處理的結果通常有三種情況。處理成功，處理失敗（處理的時候發生了錯誤）或者是拒絕去處理。在拒絕處理的情況下，這個location的處理就會由默認的handler來進行處理。例如，當在請求一個靜態檔的時候，如果你關聯一個handler到這個location上，但是拒絕處理，就會由默認的ngx_http_static_module模組進行處理，該模組是一個典型的handler。


模組的基本結構
-----------------------

在這一節我們將會對通常的模組開發過程中，每個模組所包含的一些常用的部分進行說明。這些部分有些是必須的，有些不是必須的。同時這裏所列出的這些東西對於其他類型的模組，例如filter模組等也都是相同的。


模組配置結構
~~~~~~~~~~~~~~~~~~

基本上每個模組都會提供一些配置指令，以便於用戶可以通過配置來控制該模組的行為。那麼這些配置資訊怎麼存儲呢？那就需要定義該模組的配置結構來進行存儲。

大家都知道Nginx的配置資訊分成了幾個scope，這就是main, server, 以及location。同樣的每個模組提供的配置指令也可以出現在這幾個scope裏。那對於這三個scope的配置資訊，每個模組就需要定義三個不同的資料結構去進行存儲。當然，不是每個模組都會在這三個scope都提供配置指令的。那麼也就不一定每個模組都需要定義三個資料結構去存儲這些配置資訊了。視模組的實現而言，需要幾個就定義幾個。

有一點需要特別注意的就是，在模組的開發過程中，我們最好使用nginx原有的命名習慣。這樣跟原代碼的契合度更高，看起來也更舒服。

對於模組配置資訊的定義，命名習慣是ngx_http_<module name>_(main|srv|loc)_conf_t。這裏有個例子，就是從我們後面將要展示給大家的hello module中截取的。

.. code-block:: none  

    typedef struct
    {
        ngx_str_t hello_string;
        ngx_int_t hello_counter;
    }ngx_http_hello_loc_conf_t;



模組配置指令
~~~~~~~~~~~~~~~~~~


一個模組的配置指令是定義在一個靜態陣列中的。同樣地，我們來看一下從hello module中截取的模組配置指令的定義。 

.. code-block:: none
 
    static ngx_command_t ngx_http_hello_commands[] = {
       { 
            ngx_string("hello_string"),
            NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS|NGX_CONF_TAKE1,
            ngx_http_hello_string,
            NGX_HTTP_LOC_CONF_OFFSET,
            offsetof(ngx_http_hello_loc_conf_t, hello_string),
            NULL },
     
        { 
            ngx_string("hello_counter"),
            NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
            ngx_http_hello_counter,
            NGX_HTTP_LOC_CONF_OFFSET,
            offsetof(ngx_http_hello_loc_conf_t, hello_counter),
            NULL },               
    
        ngx_null_command
    };


其實看這個定義，就基本能看出來一些資訊。例如，我們是定義了兩個配置指令，一個是叫hello_string，可以接受一個參數，或者是沒有參數。另外一個是hello_counter的參數。除此之外，似乎看起來有點迷惑。沒有關係，我們來詳細看一下ngx_command_t，一旦我們瞭解這個結構的詳細資訊，那麼我相信上述這個定義所表達的所有資訊就不言自明瞭。

ngx_command_t的定義，位於src/core/ngx_conf_file.h中。 

.. code-block:: none

    struct ngx_command_s {
        ngx_str_t             name;
        ngx_uint_t            type;
        char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
        ngx_uint_t            conf;
        ngx_uint_t            offset;
        void                 *post;
    };
    

:name: 配置指令的名稱。

:type: 該配置的類型，其實更準確一點說，是該配置指令屬性的集合。nginx提供了很多預定義的屬性值（一些巨集定義），通過邏輯或運算符可組合在一起，形成對這個配置指令的詳細的說明。下面列出可在這裏使用的預定義屬性值及說明。


*   NGX_CONF_NOARGS：配置指令不接受任何參數。
*   NGX_CONF_TAKE1：配置指令接受1個參數。
*   NGX_CONF_TAKE2：配置指令接受2個參數。
*   NGX_CONF_TAKE3：配置指令接受3個參數。
*   NGX_CONF_TAKE4：配置指令接受4個參數。
*   NGX_CONF_TAKE5：配置指令接受5個參數。
*   NGX_CONF_TAKE6：配置指令接受6個參數。
*   NGX_CONF_TAKE7：配置指令接受7個參數。

    可以組合多個屬性，比如一個指令即可以不填參數，也可以接受1個或者2個參數。那麼就是NGX_CONF_NOARGS|NGX_CONF_TAKE1|NGX_CONF_TAKE2。如果寫上面三個屬性在一起，你覺得麻煩，那麼沒有關係，nginx提供了一些定義，使用起來更簡潔。

*   NGX_CONF_TAKE12：配置指令接受1個或者2個參數。
*   NGX_CONF_TAKE13：配置指令接受1個或者3個參數。
*   NGX_CONF_TAKE23：配置指令接受2個或者3個參數。
*   NGX_CONF_TAKE123：配置指令接受1個或者2個或者3參數。
*   NGX_CONF_TAKE1234：配置指令接受1個或者2個或者3個或者4個參數。
*   NGX_CONF_1MORE：配置指令接受至少一個參數。
*   NGX_CONF_2MORE：配置指令接受至少兩個參數。
*   NGX_CONF_MULTI: 配置指令可以接受多個參數，即個數不定。
    
    
*   NGX_CONF_BLOCK：配置指令可以接受的值是一個配置資訊塊。也就是一對大括弧括起來的內容。裏面可以再包括很多的配置指令。比如常見的server指令就是這個屬性的。
*   NGX_CONF_FLAG：配置指令可以接受的值是"on"或者"off"，最終會被轉成bool值。
*   NGX_CONF_ANY：配置指令可以接受的任意的參數值。一個或者多個，或者"on"或者"off"，或者是配置塊。
    
    最後要說明的是，無論如何，nginx的配置指令的參數個數不可以超過NGX_CONF_MAX_ARGS個。目前這個值被定義為8，也就是不能超過8個參數值。
    
    下面介紹一組說明配置指令可以出現的位置的屬性。
*   NGX_DIRECT_CONF：可以出現在配置檔中最外層。例如已經提供的配置指令daemon，master_process等。
*   NGX_MAIN_CONF: http、mail、events、error_log等。
*   NGX_ANY_CONF: 該配置指令可以出現在任意配置級別上。
    
    對於我們編寫的大多數模組而言，都是在處理http相關的事情，也就是所謂的都是NGX_HTTP_MODULE，對於這樣類型的模組，其配置可能出現的位置也是分為直接出現在http裏面，以及其他位置。
*   NGX_HTTP_MAIN_CONF: 可以直接出現在http配置指令裏。
*   NGX_HTTP_SRV_CONF: 可以出現在http裏面的server配置指令裏。
*   NGX_HTTP_LOC_CONF: 可以出現在http裏面的location配置指令裏。
*   NGX_HTTP_UPS_CONF: 可以出現在http裏面的upstream配置指令裏。
*   NGX_HTTP_SIF_CONF: 可以出現在http裏面的server配置指令裏的if語句所在的block中。
*   NGX_HTTP_LIF_CONF: 可以出現在http裏面的limit_except指令的block中。


:set: 這是一個函數指標，當nginx在解析配置的時候，如果遇到這個配置指令，將會把讀取到的值傳遞給這個函數進行分解處理。因為具體每個配置指令的值如何處理，只有定義這個配置指令的人是最清楚的。來看一些這個函數指標要求的函數原型。

.. code-block:: none

    char *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);

先看該函數的返回值，處理成功時，返回NGX_OK，否則返回NGX_CONF_ERROR或者是一個自定義的錯誤資訊的字串。

在看一下這個函數被調用的時候，傳入的三個參數。

*   cf: 該參數裏面保存裏讀取到的配置資訊的原始字串以及相關的一些資訊。特別注意的是這個參數的args欄位是一個ngx_str_t類型的陣列，每個陣列元素。該陣列的首個元素是這個配置指令本身的字串，第二個元素是首個參數，第三個元素是第二個參數，依次類推。

*   cmd: 這個配置指令對應的ngx_command_t結構。

*   conf: 就是定義的存儲這個配置值的結構體，比如在上面展示的那個ngx_http_hello_loc_conf_t。當解析這個hello_string變數的時候，傳入的conf就指向一個ngx_http_hello_loc_conf_t類型的變數。用戶在處理的時候可以使用類型轉換，轉換成自己知道的類型，再進行欄位的賦值。



為了更加方便的實現對配置指令參數的讀取，nginx已經默認提供了對一些標準類型的參數進行讀取的函數，可以直接賦值個set欄位使用。下面來看一下這些已經實現的set類型函數。


*   ngx_conf_set_flag_slot： 讀取NGX_CONF_FLAG類型的參數。
*   ngx_conf_set_str_slot:讀取字串類型的參數。
*   ngx_conf_set_str_array_slot: 讀取字串陣列類型的參數。
*   ngx_conf_set_keyval_slot： 讀取鍵值對類型的參數。
*   ngx_conf_set_num_slot: 讀取整數類型(有符號整數ngx_int_t)的參數。
*   ngx_conf_set_size_slot:讀取size_t類型的參數，也就是無符號數。
*   ngx_conf_set_off_slot: 讀取off_t類型的參數。
*   ngx_conf_set_msec_slot: 讀取毫秒值類型的參數。
*   ngx_conf_set_sec_slot: 讀取秒值類型的參數。
*   ngx_conf_set_bufs_slot： 讀取的參數值是2個，一個是buf的個數，一個是buf的大小。例如： output_buffers 1 128k;
*   ngx_conf_set_enum_slot: 讀取枚舉類型的參數，將其轉換成整數ngx_uint_t類型。
*   ngx_conf_set_bitmask_slot: 讀取參數的值，並將這些參數的值以bit位元的形式存儲。例如：HttpDavModule模組的dav_methods指令。


:conf: 該欄位被NGX_HTTP_MODULE類型模組所用 (我們編寫的基本上都是NGX_HTTP_MOUDLE，只有一些nginx核心模組是非NGX_HTTP_MODULE)，該欄位指定當前配置項存儲的記憶體位置。實際上是使用哪個記憶體池的問題。因為http模組對所有http模組所要保存的配置資訊，劃分了main, server和location三個地方進行存儲，每個地方都有一個記憶體池用來分配存儲這些資訊的記憶體。這裏可能的值為 NGX_HTTP_MAIN_CONF_OFFSET、NGX_HTTP_SRV_CONF_OFFSET或NGX_HTTP_LOC_CONF_OFFSET。當然也可以直接置為0，就是NGX_HTTP_MAIN_CONF_OFFSET。

:offset: 指定該配置項值的精確存放位置，一般指定為某一個結構體變數的欄位偏移。因為對於配置資訊的存儲，一般我們都是定義個結構體來存儲的。那麼比如我們定義了一個結構體A，該項配置的值需要存儲到該結構體的b欄位。那麼在這裏就可以填寫為offsetof(A, b)。對於有些配置項，它的值不需要保存或者是需要保存到更為複雜的結構中時，這裏可以設置為0。

:post: 該欄位存儲一個指標。可以指向任何一個在讀取配置過程中需要的資料，以便於進行配置讀取的處理。大多數時候，都不需要，所以簡單地設為0即可。




看到這裏，應該就比較清楚了。ngx_http_hello_commands這個陣列每5個元素為一組，用來描述一個配置項的所有情況。那麼如果有多個配置項，只要按照需要再增加5個對應的元素對新的配置項進行說明。

**需要注意的是，就是在ngx_http_hello_commands這個陣列定義的最後，都要加一個ngx_null_command作為結尾。** 


模組上下文結構
~~~~~~~~~~~~~~~~~~

這是一個ngx_http_module_t類型的靜態變數。這個變數實際上是提供一組回調函數指標，這些函數有在創建存儲配置資訊的物件的函數，也有在創建前和創建後會調用的函數。這些函數都將被nginx在合適的時間進行調用。

.. code-block:: none

    typedef struct {
        ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
        ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);
    
        void       *(*create_main_conf)(ngx_conf_t *cf);
        char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);
    
        void       *(*create_srv_conf)(ngx_conf_t *cf);
        char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);
    
        void       *(*create_loc_conf)(ngx_conf_t *cf);
        char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
    } ngx_http_module_t; 



:preconfiguration: 在創建和讀取該模組的配置資訊之前被調用。

:postconfiguration: 在創建和讀取該模組的配置資訊之後被調用。

:create_main_conf: 調用該函數創建本模組位於http block的配置資訊存儲結構。該函數成功的時候，返回創建的配置物件。失敗的話，返回NULL。

:init_main_conf: 調用該函數初始化本模組位於http block的配置資訊存儲結構。該函數成功的時候，返回NGX_CONF_OK。失敗的話，返回NGX_CONF_ERROR或錯誤字串。

:create_srv_conf: 調用該函數創建本模組位於http server block的配置資訊存儲結構，每個server block會創建一個。該函數成功的時候，返回創建的配置物件。失敗的話，返回NULL。

:merge_srv_conf: 因為有些配置指令即可以出現在http block，也可以出現在http server block中。那麼遇到這種情況，每個server都會有自己存儲結構來存儲該server的配置，但是在這種情況下當在http block中的配置與server block中的配置資訊衝突的時候，就需要調用此函數進行合併，該函數並非必須提供，當預計到絕對不會發生需要合併的情況的時候，就無需提供。當然為了安全期間還是建議提供。該函數成功的時候，返回NGX_CONF_OK。失敗的話，返回NGX_CONF_ERROR或錯誤字串。

:create_loc_conf: 調用該函數創建本模組位於location block的配置資訊存儲結構。每個在配置中指明的location創建一個。該函數成功的時候，返回創建的配置物件。失敗的話，返回NULL。

:merge_loc_conf: 與merge_srv_conf類似，這個也是進行配置值合併的地方。該函數成功的時候，返回NGX_CONF_OK。失敗的話，返回NGX_CONF_ERROR或錯誤字串。

Nginx裏面的配置資訊都是上下一層層的嵌套的，對於具體某個location的話，對於同一個配置，如果自己這裏沒有定義，那麼就使用上層的配置，否則是用自己的配置。

這些配置資訊一般默認都應該設為一個未初始化的值，針對這個需求，Nginx定義了一系列的巨集定義來代表個中配置所對應資料類型的未初始化值，如下：

.. code-block:: none

    #define NGX_CONF_UNSET       -1
    #define NGX_CONF_UNSET_UINT  (ngx_uint_t) -1
    #define NGX_CONF_UNSET_PTR   (void *) -1
    #define NGX_CONF_UNSET_SIZE  (size_t) -1
    #define NGX_CONF_UNSET_MSEC  (ngx_msec_t) -1

又因為對於配置項的合併，邏輯都類似，也就是前面已經說過的，如果在本層次已經配置了，也就是配置項的值已經被讀取進來了（那麼這些配置項的值就不會等於上面已經定義的那些UNSET的值），就使用本層次的值作為定義合併的結果，否則，使用上層的值，如果上層的值也是這些UNSET類的值，那就複製為預設值，否則就是用上層的值作為合併的結果。對於這樣類似的操作，Nginx定義了一些宏操作來做這些事情，我們來看其中一個的定義。

.. code-block:: none

    #define ngx_conf_merge_uint_value(conf, prev, default)                       \
        if (conf == NGX_CONF_UNSET_UINT) {                                       \
            conf = (prev == NGX_CONF_UNSET_UINT) ? default : prev;               \
        }
    

顯而易見，這個邏輯確實比較簡單，所以其他的巨集定義也類似，我們就列具其中的一部分吧。

.. code-block:: none

    ngx_conf_merge_value
    ngx_conf_merge_ptr_value
    ngx_conf_merge_uint_value
    ngx_conf_merge_msec_value
    ngx_conf_merge_sec_value


等等。

  


下面來看一下hello模組的模組上下文的定義，加深一下印象。 

.. code-block:: none

    static ngx_http_module_t ngx_http_hello_module_ctx = {
        NULL,                          /* preconfiguration */
        ngx_http_hello_init,           /* postconfiguration */
     
        NULL,                          /* create main configuration */
        NULL,                          /* init main configuration */
     
        NULL,                          /* create server configuration */
        NULL,                          /* merge server configuration */
     
        ngx_http_hello_create_loc_conf, /* create location configuration */
        NULL                        /* merge location configuration */
    };


**注意：這裏並沒有提供merge_loc_conf函數，因為我們這個模組的配置指令已經確定只出現在NGX_HTTP_LOC_CONF中這一個level上，不會發生需要合併的情況。**




模組的定義
~~~~~~~~~~~~~~~~~~

對於開發一個模組來說，我們都需要定義一個ngx_module_t類型的變數來說明這個模組本身的資訊，從某種意義上來說，這是這個模組最重要的一個資訊，它告訴了nginx這個模組的一些資訊，上面定義的配置資訊，還有模組上下文資訊，都是通過這個結構來告訴nginx系統的，也就是載入模組的上層代碼，都需要通過定義的這個結構，來獲取這些資訊。

我們來看一下hello模組的模組定義。

.. code-block:: none

    ngx_module_t ngx_http_hello_module = {
        NGX_MODULE_V1,
        &ngx_http_hello_module_ctx,    /* module context */
        ngx_http_hello_commands,       /* module directives */
        NGX_HTTP_MODULE,               /* module type */
        NULL,                          /* init master */
        NULL,                          /* init module */
        NULL,                          /* init process */
        NULL,                          /* init thread */
        NULL,                          /* exit thread */
        NULL,                          /* exit process */
        NULL,                          /* exit master */
        NGX_MODULE_V1_PADDING
    };


模組可以提供一些回調函數給nginx，當nginx在創建進程線程或者結束進程線程時進行調用。但大多數模組在這些時刻並不需奧做寫什麼事情，所以都簡單賦值為NULL。






handler模組的基本結構
-----------------------

除了上一節介紹的模組的基本結構以外，handler模組必須提供一個真正的處理函數，這個函數負責對來自用戶端請求的真正處理。這個函數的處理，即可以選擇自己直接生成內容，也可以選擇拒絕處理，由後續的handler去進行處理，或者是選擇丟給後續的filter進行處理。來看一下這個函數的原型申明。

typedef ngx_int_t (\*ngx_http_handler_pt)(ngx_http_request_t  \*r);

r是http請求。裏面包含請求所有的資訊，這裏不相信說明了，可以參考別的章節的介紹。
該函數處理成功返回NGX_OK，處理發生錯誤返回NGX_ERROR，拒絕處理（留給後續的handler進行處理）返回NGX_DECLINE。
返回NGX_OK也就代表給用戶端的回應已經生成好了，否則返回NGX_OK就發生錯誤了。



handler模組的掛載
-----------------------


按處理階段掛載
~~~~~~~~~~~~~~~~~~

為了更精細地控制對於用戶端請求的處理過程，nginx把這個處理過程劃分成了11個階段。他們從前到後，依次列舉如下：

:NGX_HTTP_POST_READ_PHASE:	讀取請求內容階段
:NGX_HTTP_SERVER_REWRITE_PHASE:	Server請求位址重寫階段
:NGX_HTTP_FIND_CONFIG_PHASE:	配置查找階段:
:NGX_HTTP_REWRITE_PHASE:	Location請求位址重寫階段
:NGX_HTTP_POST_REWRITE_PHASE:	請求位址重寫提交階段
:NGX_HTTP_PREACCESS_PHASE:	訪問許可權檢查準備階段
:NGX_HTTP_ACCESS_PHASE:	訪問許可權檢查階段
:NGX_HTTP_POST_ACCESS_PHASE:	訪問許可權檢查提交階段
:NGX_HTTP_TRY_FILES_PHASE:	配置項try_files處理階段  
:NGX_HTTP_CONTENT_PHASE:	內容產生階段
:NGX_HTTP_LOG_PHASE:	日誌模組處理階段


一般情況下，我們自定義的模組，大多數是掛載在NGX_HTTP_CONTENT_PHASE階段的。掛載的動作一般是現在模組上下文調用的postconfiguration函數中。

**注意：有幾個階段是特例，它不調用掛載地任何的handler，也就是你就不用掛載到這幾個階段了：**

- NGX_HTTP_FIND_CONFIG_PHASE
- NGX_HTTP_POST_ACCESS_PHASE
- NGX_HTTP_POST_REWRITE_PHASE
- NGX_HTTP_TRY_FILES_PHASE


所以其實真正是有6個phase你可以去掛載handler。

掛載的代碼如下（摘自hello module）:

.. code-block:: none

	static ngx_int_t
	ngx_http_hello_init(ngx_conf_t *cf)
	{
		ngx_http_handler_pt        *h;
		ngx_http_core_main_conf_t  *cmcf;

		cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

		h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
		if (h == NULL) {
			return NGX_ERROR;
		}

		*h = ngx_http_hello_handler;

		return NGX_OK;
	}


    
使用這種方式掛載的handler也被稱為 **content phase handlers**。

按需掛載
~~~~~~~~~~~~~~~~~~~~~~~

以這種方式掛載的handler也被稱為 **content handler**。

一個請求進來以後，nginx按照從NGX_HTTP_POST_READ_PHASE開始的階段，去依次執行每個階段的所有handler。等到執行到 NGX_HTTP_CONTENT_PHASE階段的時候，如果這個location對應的有一個content handler，那麼就去執行這個content handler。否則去依次執行NGX_HTTP_CONTENT_PHASE階段掛載的所有content phase handlers，直到某個函數處理返回NGX_OK或者NGX_ERROR。

換句話說，如果某個location在處理到NGX_HTTP_CONTENT_PHASE階段的時候，如果有content handler，那麼所有的掛載的content phase handlers都不會被執行了。

使用這個方法掛載上去的handler，必須在NGX_HTTP_CONTENT_PHASE階段才能執行到。如果你想自己的handler要被更早的執行到的話，那就不要使用這種掛載方式。

另外要提一下，在什麼情況會使用這種方式來掛載。一般就是某個模組如果對某個location進行了處理以後，發現符合自己處理的邏輯，而且也沒有必要再調用NGX_HTTP_CONTENT_PHASE階段的其他handler進行處理的時候，就動態掛載上這個handler。

好了，下面看一下這種掛載方式的具體代碼（摘自Emiller's Guide To Nginx Module Development）。

.. code-block:: none

	static char *
	ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
	{
		ngx_http_core_loc_conf_t  *clcf;

		clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
		clcf->handler = ngx_http_circle_gif_handler;

		return NGX_CONF_OK;
	}



handler的編寫步驟
-----------------------

好，到了這裏，讓我們稍微整理一下思路，回顧一下實現一個handler的步驟:

1. 編寫模組基本結構。
2. 實現handler的掛載函數。
#. 編寫handler處理函數。

看起來不是那麼難，對吧？還是那句老話，世上無難事，只怕有心人!

hello handler 模組
-------------------------

我們在前面已經看到了這個hello handler module的部分重要的結構。現在我們完整的介紹一下這個示例模組的功能和代碼。

該模組提供了2個配置指令，僅可以出現在location指令的block中。這兩個指令是hello_string, 該參數接受一個參數來設置顯示的字串。如果沒有跟參數，那麼就使用默認的字串作為回應字串。

另一個參數是hello_counter，如果設置為on，則會在響應的字串後面追加Visited Times:的字樣，以統計請求的次數。

這裏有兩點注意一下：

1. 對於flag類型的配置指令，當值為off的時候，使用ngx_conf_set_flag_slot函數，會轉化為0，為on，則轉化為非0。
2. 另外一個是，我提供了merge_loc_conf函數，但是卻沒有設置到模組的上下文定義中。這樣有一個缺點，就是如果一個指令沒有出現在配置檔中的時候，配置資訊中的值，將永遠會保持在create_loc_conf中的初始化的值。那如果，在類似create_loc_conf這樣的函數中，對創建出來的配置資訊的值，沒有設置為合理的值的話，後面用戶又沒有配置，就會出現問題。
    
下面來完整的給出ngx_http_hello_module模組的完整代碼。

.. code-block:: none

	#include <ngx_config.h>
	#include <ngx_core.h>
	#include <ngx_http.h>


	typedef struct
	{
		ngx_str_t hello_string;
		ngx_int_t hello_counter;
	}ngx_http_hello_loc_conf_t;

	static ngx_int_t ngx_http_hello_init(ngx_conf_t *cf);

	static void *ngx_http_hello_create_loc_conf(ngx_conf_t *cf);

	static char *ngx_http_hello_string(ngx_conf_t *cf, ngx_command_t *cmd,
		void *conf);
	static char *ngx_http_hello_counter(ngx_conf_t *cf, ngx_command_t *cmd,
		void *conf);
	 
	static ngx_command_t ngx_http_hello_commands[] = {
	   { 
			ngx_string("hello_string"),
			NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS|NGX_CONF_TAKE1,
			ngx_http_hello_string,
			NGX_HTTP_LOC_CONF_OFFSET,
			offsetof(ngx_http_hello_loc_conf_t, hello_string),
			NULL },
	 
		{ 
			ngx_string("hello_counter"),
			NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
			ngx_http_hello_counter,
			NGX_HTTP_LOC_CONF_OFFSET,
			offsetof(ngx_http_hello_loc_conf_t, hello_counter),
			NULL },               

		ngx_null_command
	};
	 

	/* 
	static u_char ngx_hello_default_string[] = "Default String: Hello, world!";
	*/
	static int ngx_hello_visited_times = 0; 
	 
	static ngx_http_module_t ngx_http_hello_module_ctx = {
		NULL,                          /* preconfiguration */
		ngx_http_hello_init,           /* postconfiguration */
	 
		NULL,                          /* create main configuration */
		NULL,                          /* init main configuration */
	 
		NULL,                          /* create server configuration */
		NULL,                          /* merge server configuration */
	 
		ngx_http_hello_create_loc_conf, /* create location configuration */
		NULL                            /* merge location configuration */
	};
	 
	 
	ngx_module_t ngx_http_hello_module = {
		NGX_MODULE_V1,
		&ngx_http_hello_module_ctx,    /* module context */
		ngx_http_hello_commands,       /* module directives */
		NGX_HTTP_MODULE,               /* module type */
		NULL,                          /* init master */
		NULL,                          /* init module */
		NULL,                          /* init process */
		NULL,                          /* init thread */
		NULL,                          /* exit thread */
		NULL,                          /* exit process */
		NULL,                          /* exit master */
		NGX_MODULE_V1_PADDING
	};
	 
	 
	static ngx_int_t
	ngx_http_hello_handler(ngx_http_request_t *r)
	{
		ngx_int_t    rc;
		ngx_buf_t   *b;
		ngx_chain_t  out;
		ngx_http_hello_loc_conf_t* my_conf;
		u_char ngx_hello_string[1024] = {0};
		ngx_uint_t content_length = 0;
		
		ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "ngx_http_hello_handler is called!");
		
		my_conf = ngx_http_get_module_loc_conf(r, ngx_http_hello_module);
		if (my_conf->hello_string.len == 0 )
		{
			ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "hello_string is empty!");
			return NGX_DECLINED;
		}
		
		
		if (my_conf->hello_counter == NGX_CONF_UNSET
			|| my_conf->hello_counter == 0)
		{
			ngx_sprintf(ngx_hello_string, "%s", my_conf->hello_string.data);
		}
		else
		{
			ngx_sprintf(ngx_hello_string, "%s Visited Times:%d", my_conf->hello_string.data, 
				++ngx_hello_visited_times);
		}
		ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "hello_string:%s", ngx_hello_string);
		content_length = ngx_strlen(ngx_hello_string);
		 
		/* we response to 'GET' and 'HEAD' requests only */
		if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
			return NGX_HTTP_NOT_ALLOWED;
		}
	 
		/* discard request body, since we don't need it here */
		rc = ngx_http_discard_request_body(r);
	 
		if (rc != NGX_OK) {
			return rc;
		}
	 
		/* set the 'Content-type' header */
		/*
		r->headers_out.content_type_len = sizeof("text/html") - 1;
		r->headers_out.content_type.len = sizeof("text/html") - 1;
		r->headers_out.content_type.data = (u_char *)"text/html";*/
		ngx_str_set(&r->headers_out.content_type, "text/html");
		
	 
		/* send the header only, if the request type is http 'HEAD' */
		if (r->method == NGX_HTTP_HEAD) {
			r->headers_out.status = NGX_HTTP_OK;
			r->headers_out.content_length_n = content_length;
	 
			return ngx_http_send_header(r);
		}
	 
		/* allocate a buffer for your response body */
		b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
		if (b == NULL) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}
	 
		/* attach this buffer to the buffer chain */
		out.buf = b;
		out.next = NULL;
	 
		/* adjust the pointers of the buffer */
		b->pos = ngx_hello_string;
		b->last = ngx_hello_string + content_length;
		b->memory = 1;    /* this buffer is in memory */
		b->last_buf = 1;  /* this is the last buffer in the buffer chain */
	 
		/* set the status line */
		r->headers_out.status = NGX_HTTP_OK;
		r->headers_out.content_length_n = content_length;
	 
		/* send the headers of your response */
		rc = ngx_http_send_header(r);
	 
		if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
			return rc;
		}
	 
		/* send the buffer chain of your response */
		return ngx_http_output_filter(r, &out);
	}

	static void *ngx_http_hello_create_loc_conf(ngx_conf_t *cf)
	{
		ngx_http_hello_loc_conf_t* local_conf = NULL;
		local_conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_hello_loc_conf_t));
		if (local_conf == NULL)
		{
			return NULL;
		}
		
		ngx_str_null(&local_conf->hello_string);
		local_conf->hello_counter = NGX_CONF_UNSET;
		
		return local_conf;
	} 

	/*
	static char *ngx_http_hello_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
	{
		ngx_http_hello_loc_conf_t* prev = parent;
		ngx_http_hello_loc_conf_t* conf = child;
		
		ngx_conf_merge_str_value(conf->hello_string, prev->hello_string, ngx_hello_default_string);
		ngx_conf_merge_value(conf->hello_counter, prev->hello_counter, 0);
		
		return NGX_CONF_OK;
	}*/

	static char *
	ngx_http_hello_string(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
	{
		ngx_http_core_loc_conf_t *clcf;
		ngx_http_hello_loc_conf_t* local_conf;
		 
		clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
		
		local_conf = conf;
		char* rv = ngx_conf_set_str_slot(cf, cmd, conf);

		ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "hello_string:%s", local_conf->hello_string.data);
		
		return rv;
	}


	static char *ngx_http_hello_counter(ngx_conf_t *cf, ngx_command_t *cmd,
		void *conf)
	{
		ngx_http_hello_loc_conf_t* local_conf;
		ngx_http_core_loc_conf_t *clcf;

		clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
		
		local_conf = conf;
		
		char* rv = NULL;
		
		rv = ngx_conf_set_flag_slot(cf, cmd, conf);
		
		
		ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "hello_counter:%d", local_conf->hello_counter);
		return rv;    
	}

	static ngx_int_t
	ngx_http_hello_init(ngx_conf_t *cf)
	{
		ngx_http_handler_pt        *h;
		ngx_http_core_main_conf_t  *cmcf;

		cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

		h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
		if (h == NULL) {
			return NGX_ERROR;
		}

		*h = ngx_http_hello_handler;

		return NGX_OK;
	}


通過上面一些介紹，我相信大家都能對整個程式有一個比較好的理解。唯一可能感覺有些理解困難的地方在於ngx_http_hello_handler函數裏面產生和設置輸出。但其實大家在本書的前面的相關章節都可以看到對ngx_buf_t和request等相關資料結構的說明。如果仔細看了這些地方的說明的話，應該對這裏代碼的實現就比較容易理解了。因此，這裏不再贅述解釋。



handler模組的編譯和使用
-------------------------


config檔的編寫
~~~~~~~~~~~~~~~~~~

對於開發一個模組，我們是需要把這個模組的C代碼組織到一個目錄裏，同時需要編寫一個config檔。這個config檔的內容就是告訴nginx的編譯腳本，該如何進行編譯。我們來看一下hello handler module的config檔的內容，然後再做解釋。

.. code-block:: none

	ngx_addon_name=ngx_http_hello_module
	HTTP_MODULES="$HTTP_MODULES ngx_http_hello_module"
	NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_module.c"

其實檔很簡單，幾乎不需要做什麼解釋。大家一看都懂了。唯一需要說明的是，如果這個模組的實現有多個原始檔案，那麼都在NGX_ADDON_SRCS這個變數裏，依次寫進去就可以。


編譯
~~~~~~~~~~~~~~~~~~

對於模組的編譯，nginx並不像apache一樣，提供了單獨的編譯工具，可以在沒有nginx源代碼的情況下來單獨編譯一個模組的代碼。nginx必須去到nginx的源代碼目錄裏，通過configure指令的參數，來進行編譯。下面看一下hello module的configure指令：
        
./configure --prefix=/usr/local/nginx-1.3.1 --add-module=/home/jizhao/open_source/book_module

我寫的這個示例模組的代碼和config檔都放在/home/jizhao/open_source/book_module這個目錄下。所以一切都很明瞭，也沒什麼好說的了。


使用
~~~~~~~~~~~~~~~~~~

使用一個模組需要根據這個模組定義的配置指令來做。比如我們這個簡單的hello handler module的使用就很簡單。在我的測試伺服器的配置檔裏，就是在http裏面的默認的server裏面加入如下的配置：

.. code-block:: none

	location /test {
			hello_string jizhao;
			hello_counter on;
	}

當我們訪問這個位址的時候, lynx http://127.0.0.1/test的時候，就可以看到返回的結果。

jizhao Visited Times:1

當然你訪問多次，這個次數是會增加的。

部分handler模組的分析
-----------------------


http access module 
~~~~~~~~~~~~~~~~~~

該模組的代碼位於src/http/modules/ngx_http_access_module.c中。該模組的作用是提供對於特定host的用戶端的訪問控制。可以限定特定host的用戶端對於服務端全部，或者某個server，或者是某個location的訪問。
該模組的實現非常簡單，總共也就只有幾個函數。

.. code-block:: none

	static ngx_int_t ngx_http_access_handler(ngx_http_request_t *r);
	static ngx_int_t ngx_http_access_inet(ngx_http_request_t *r,
		ngx_http_access_loc_conf_t *alcf, in_addr_t addr);
	#if (NGX_HAVE_INET6)
	static ngx_int_t ngx_http_access_inet6(ngx_http_request_t *r,
		ngx_http_access_loc_conf_t *alcf, u_char *p);
	#endif
	static ngx_int_t ngx_http_access_found(ngx_http_request_t *r, ngx_uint_t deny);
	static char *ngx_http_access_rule(ngx_conf_t *cf, ngx_command_t *cmd,
		void *conf);
	static void *ngx_http_access_create_loc_conf(ngx_conf_t *cf);
	static char *ngx_http_access_merge_loc_conf(ngx_conf_t *cf,
		void *parent, void *child);
	static ngx_int_t ngx_http_access_init(ngx_conf_t *cf);

對於與配置相關的幾個函數都不需要做解釋了，需要提一下的是函數ngx_http_access_init，該函數在實現上把本模組掛載到了NGX_HTTP_ACCESS_PHASE階段的handler上，從而使自己的被調用時機發生在了NGX_HTTP_CONTENT_PHASE等階段前。因為進行用戶端位址的限制檢查，根本不需要等到這麼後面。

另外看一下這個模組的主處理函數ngx_http_access_handler。這個函數的邏輯也非常簡單，主要是根據用戶端位址的類型，來分別選則ipv4類型的處理函數ngx_http_access_inet還是ipv6類型的處理函數ngx_http_access_inet6。

而這個兩個處理函數內部也非常簡單，就是迴圈檢查每個規則，檢查是否有匹配的規則，如果有就返回匹配的結果，如果都沒有匹配，就默認拒絕。  


http static module 
~~~~~~~~~~~~~~~~~~

從某種程度上來說，此模組可以算的上是“最正宗的”，“最古老”的content handler。因為本模組的作用就是讀取磁片上的靜態檔，並把檔內容作為產生的輸出。在Web技術發展的早期，只有靜態頁面，沒有服務端腳本來動態生成HTML的時候。恐怕開發個Web伺服器的時候，第一個要開發就是這樣一個content handler。

http static module的代碼位於src\http\modules\ngx_http_static_module.c中，總共只有兩百多行近三百行。可以說是非常短小。

我們首先來看一下該模組的模組上下文的定義。

.. code-block:: none

	ngx_http_module_t  ngx_http_static_module_ctx = {
		NULL,                                  /* preconfiguration */
		ngx_http_static_init,                  /* postconfiguration */

		NULL,                                  /* create main configuration */
		NULL,                                  /* init main configuration */

		NULL,                                  /* create server configuration */
		NULL,                                  /* merge server configuration */

		NULL,                                  /* create location configuration */
		NULL                                   /* merge location configuration */
	};

是非常的簡潔吧，連任何與配置相關的函數都沒有。對了，因為該模組沒有提供任何配置指令。大家想想也就知道了，這個模組做的事情實在是太簡單了，也確實沒什麼好配置的。唯一需要調用的函數是一個ngx_http_static_init函數。好了，來看一下這個函數都幹了寫什麼。

.. code-block:: none

	static ngx_int_t
	ngx_http_static_init(ngx_conf_t *cf)
	{
		ngx_http_handler_pt        *h;
		ngx_http_core_main_conf_t  *cmcf;

		cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

		h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
		if (h == NULL) {
			return NGX_ERROR;
		}

		*h = ngx_http_static_handler;

		return NGX_OK;
	}

僅僅是掛載這個handler到NGX_HTTP_CONTENT_PHASE處理階段。簡單吧？

下面我們就看一下這個模組最核心的處理邏輯所在的ngx_http_static_handler函數。該函數大概占了這個模組代碼量的百分之八九十。

.. code-block:: none

	static ngx_int_t
	ngx_http_static_handler(ngx_http_request_t *r)
	{
		u_char                    *last, *location;
		size_t                     root, len;
		ngx_str_t                  path;
		ngx_int_t                  rc;
		ngx_uint_t                 level;
		ngx_log_t                 *log;
		ngx_buf_t                 *b;
		ngx_chain_t                out;
		ngx_open_file_info_t       of;
		ngx_http_core_loc_conf_t  *clcf;

		if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD|NGX_HTTP_POST))) {
			return NGX_HTTP_NOT_ALLOWED;
		}

		if (r->uri.data[r->uri.len - 1] == '/') {
			return NGX_DECLINED;
		}

		log = r->connection->log;

		/*
		 * ngx_http_map_uri_to_path() allocates memory for terminating '\0'
		 * so we do not need to reserve memory for '/' for possible redirect
		 */

		last = ngx_http_map_uri_to_path(r, &path, &root, 0);
		if (last == NULL) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}

		path.len = last - path.data;

		ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0,
					   "http filename: \"%s\"", path.data);

		clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

		ngx_memzero(&of, sizeof(ngx_open_file_info_t));

		of.read_ahead = clcf->read_ahead;
		of.directio = clcf->directio;
		of.valid = clcf->open_file_cache_valid;
		of.min_uses = clcf->open_file_cache_min_uses;
		of.errors = clcf->open_file_cache_errors;
		of.events = clcf->open_file_cache_events;

		if (ngx_http_set_disable_symlinks(r, clcf, &path, &of) != NGX_OK) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}

		if (ngx_open_cached_file(clcf->open_file_cache, &path, &of, r->pool)
			!= NGX_OK)
		{
			switch (of.err) {

			case 0:
				return NGX_HTTP_INTERNAL_SERVER_ERROR;

			case NGX_ENOENT:
			case NGX_ENOTDIR:
			case NGX_ENAMETOOLONG:

				level = NGX_LOG_ERR;
				rc = NGX_HTTP_NOT_FOUND;
				break;

			case NGX_EACCES:
	#if (NGX_HAVE_OPENAT)
			case NGX_EMLINK:
			case NGX_ELOOP:
	#endif

				level = NGX_LOG_ERR;
				rc = NGX_HTTP_FORBIDDEN;
				break;

			default:

				level = NGX_LOG_CRIT;
				rc = NGX_HTTP_INTERNAL_SERVER_ERROR;
				break;
			}

			if (rc != NGX_HTTP_NOT_FOUND || clcf->log_not_found) {
				ngx_log_error(level, log, of.err,
							  "%s \"%s\" failed", of.failed, path.data);
			}

			return rc;
		}

		r->root_tested = !r->error_page;

		ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0, "http static fd: %d", of.fd);

		if (of.is_dir) {

			ngx_log_debug0(NGX_LOG_DEBUG_HTTP, log, 0, "http dir");

			ngx_http_clear_location(r);

			r->headers_out.location = ngx_palloc(r->pool, sizeof(ngx_table_elt_t));
			if (r->headers_out.location == NULL) {
				return NGX_HTTP_INTERNAL_SERVER_ERROR;
			}

			len = r->uri.len + 1;

			if (!clcf->alias && clcf->root_lengths == NULL && r->args.len == 0) {
				location = path.data + clcf->root.len;

				*last = '/';

			} else {
				if (r->args.len) {
					len += r->args.len + 1;
				}

				location = ngx_pnalloc(r->pool, len);
				if (location == NULL) {
					return NGX_HTTP_INTERNAL_SERVER_ERROR;
				}

				last = ngx_copy(location, r->uri.data, r->uri.len);

				*last = '/';

				if (r->args.len) {
					*++last = '?';
					ngx_memcpy(++last, r->args.data, r->args.len);
				}
			}

			/*
			 * we do not need to set the r->headers_out.location->hash and
			 * r->headers_out.location->key fields
			 */

			r->headers_out.location->value.len = len;
			r->headers_out.location->value.data = location;

			return NGX_HTTP_MOVED_PERMANENTLY;
		}

	#if !(NGX_WIN32) /* the not regular files are probably Unix specific */

		if (!of.is_file) {
			ngx_log_error(NGX_LOG_CRIT, log, 0,
						  "\"%s\" is not a regular file", path.data);

			return NGX_HTTP_NOT_FOUND;
		}

	#endif

		if (r->method & NGX_HTTP_POST) {
			return NGX_HTTP_NOT_ALLOWED;
		}

		rc = ngx_http_discard_request_body(r);

		if (rc != NGX_OK) {
			return rc;
		}

		log->action = "sending response to client";

		r->headers_out.status = NGX_HTTP_OK;
		r->headers_out.content_length_n = of.size;
		r->headers_out.last_modified_time = of.mtime;

		if (ngx_http_set_content_type(r) != NGX_OK) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}

		if (r != r->main && of.size == 0) {
			return ngx_http_send_header(r);
		}

		r->allow_ranges = 1;

		/* we need to allocate all before the header would be sent */

		b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
		if (b == NULL) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}

		b->file = ngx_pcalloc(r->pool, sizeof(ngx_file_t));
		if (b->file == NULL) {
			return NGX_HTTP_INTERNAL_SERVER_ERROR;
		}

		rc = ngx_http_send_header(r);

		if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
			return rc;
		}

		b->file_pos = 0;
		b->file_last = of.size;

		b->in_file = b->file_last ? 1: 0;
		b->last_buf = (r == r->main) ? 1: 0;
		b->last_in_chain = 1;

		b->file->fd = of.fd;
		b->file->name = path;
		b->file->log = log;
		b->file->directio = of.is_directio;

		out.buf = b;
		out.next = NULL;

		return ngx_http_output_filter(r, &out);
	}

首先是檢查NGX_HTTP_GET|NGX_HTTP_HEAD|NGX_HTTP_POST，對就是用戶端的請求類型，就這三種，其他一律NGX_HTTP_NOT_ALLOWED。

其次是檢查請求的url的結尾字元是不是斜杠‘/’，如果是說明請求的不是一個檔，給後續的handler去處理，比如後續的ngx_http_autoindex_handler（如果是請求的是一個目錄下面，可以列出這個目錄的檔），或者是ngx_http_index_handler（如果請求的路徑下面有個默認的index檔，直接返回index檔的內容）。

然後接下來調用了一個ngx_http_map_uri_to_path函數，該函數的作用是把請求的http協定的路徑轉化成一個檔系統的路徑。

然後根據轉化出來的具體路徑，去打開檔，打開檔的時候做了2中檢查，一種是，如果請求的檔是個symbol link，根據配置，是否允許符號鏈結，不允許返回錯誤。還有一個檢查是，如果請求的是一個名稱，是一個目錄的名字，也返回錯誤。如果都沒有檔，就讀取檔，返回內容。其實說返回內容可能不是特別準確，比較準確的說法是，把產生的內容傳遞給後續的filter去處理。


http log module
~~~~~~~~~~~~~~~~~~

該模組提供了對於每一個http請求進行記錄的功能，也就是我們見到的access.log。當然這個模組對於log提供了一些配置指令，使得可以比較方便的定制access.log。

這個模組的代碼位於src/http/modules/ngx_http_log_module.c，雖然這個模組的代碼有接近1400行，但是主要的邏輯在於對日誌本身格式啊，等細節的處理。我們在這裏進行分析主要是關注，如何編寫一個log handler的問題。

由於log handler的時候，拿到的參數也是requset這個東西，那麼也就意味著我們如果需要，可以好好研究下這個結構，把我們需要的所有資訊都記錄下來。

對於log handler，有一點特別需要注意的就是，log handler是無論如何都會被調用的，就是只要服務端接受到了一個用戶端的請求，也就是產生了一個requset物件，那麼這些個log handler的處理函數都會被調用的，就是在釋放requset的時候被調用的（ngx_http_free_request函數）。

那麼當然絕對不能忘記的就是log handler最好，也是建議被掛載在NGX_HTTP_LOG_PHASE階段。因為掛載在其他階段，有可能在某些情況下被跳過，而沒有執行到，導致你的log模組記錄的資訊不全。

還有一點要說明的是，由於nginx是允許在某個階段有多個handler模組存在的，根據其處理結果，確定是否要調用下一個handler。但是對於掛載在NGX_HTTP_LOG_PHASE階段的handler，則根本不關注這裏handler的具體處理函數的返回值，所有的都被調用。如下，位於src/http/ngx_http_request.c中的ngx_http_log_request函數。

.. code-block:: none

	static void
	ngx_http_log_request(ngx_http_request_t *r)
	{
		ngx_uint_t                  i, n;
		ngx_http_handler_pt        *log_handler;
		ngx_http_core_main_conf_t  *cmcf;

		cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

		log_handler = cmcf->phases[NGX_HTTP_LOG_PHASE].handlers.elts;
		n = cmcf->phases[NGX_HTTP_LOG_PHASE].handlers.nelts;

		for (i = 0; i < n; i++) {
			log_handler[i](r);
		}
	}

