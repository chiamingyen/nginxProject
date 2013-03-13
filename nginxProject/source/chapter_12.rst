nginx的請求處理階段 (30%)
=======================================



接收請求流程 (99%)
-----------------------



http請求格式簡介 (99%)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
首先介紹一下rfc2616中定義的http請求基本格式：

.. code-block:: none

    Request = Request-Line 
              * (( general-header         
                 | request-header          
                 | entity-header ) CRLF)  
              CRLF
              [ message-body ]

第一行是請求行（request line），用來說明請求方法，要訪問的資源以及所使用的HTTP版本：

.. code-block:: none

    Request-Line   = Method SP Request-URI SP HTTP-Version CRLF

請求方法（Method）的定義如下，其中最常用的是GET，POST方法：

.. code-block:: none

    Method = "OPTIONS" 
    | "GET" 
    | "HEAD" 
    | "POST" 
    | "PUT" 
    | "DELETE" 
    | "TRACE" 
    | "CONNECT" 
    | extension-method 
    extension-method = token

要訪問的資源由統一資源地位符URI(Uniform Resource Identifier)確定，它的一個比較通用的組成格式（rfc2396）如下：

.. code-block:: none

    <scheme>://<authority><path>?<query> 

一般來說根據請求方法（Method）的不同，請求URI的格式會有所不同，通常只需寫出path和query部分。

http版本(version)定義如下，現在用的一般為1.0和1.1版本：

.. code-block:: none

    HTTP/<major>.<minor>

請求行的下一行則是請求頭，rfc2616中定義了3種不同類型的請求頭，分別為general-header，request-header和entity-header，每種類型rfc中都定義了一些通用的頭，其中entity-header類型可以包含自定義的頭。


請求頭讀取 (99%)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

這一節介紹nginx中請求頭的解析，nginx的請求處理流程中，會涉及到2個非常重要的資料結構，ngx_connection_t和ngx_http_request_t，分別用來表示連接和請求，這2個資料結構在本書的前篇中已經做了比較詳細的介紹，沒有印象的讀者可以翻回去復習一下，整個請求處理流程從頭到尾，對應著這2個資料結構的分配，初始化，使用，重用和銷毀。

nginx在初始化階段，具體是在init process階段的ngx_event_process_init函數中會為每一個監聽套接字分配一個連接結構（ngx_connection_t），並將該連接結構的讀事件成員（read）的事件處理函數設置為ngx_event_accept，並且如果沒有使用accept互斥鎖的話，在這個函數中會將該讀事件掛載到nginx的事件處理模型上（poll或者epoll等），反之則會等到init process階段結束，在工作進程的事件處理迴圈中，某個進程搶到了accept鎖才能掛載該讀事件。

.. code-block:: none

    static ngx_int_t
    ngx_event_process_init(ngx_cycle_t *cycle)
    {
        ...

        /* 初始化用來管理所有計時器的紅黑樹 */
        if (ngx_event_timer_init(cycle->log) == NGX_ERROR) {
            return NGX_ERROR;
        }
        /* 初始化事件模型 */
        for (m = 0; ngx_modules[m]; m++) {
            if (ngx_modules[m]->type != NGX_EVENT_MODULE) {
                continue;
            }

            if (ngx_modules[m]->ctx_index != ecf->use) {
                continue;
            }

            module = ngx_modules[m]->ctx;

            if (module->actions.init(cycle, ngx_timer_resolution) != NGX_OK) {
                /* fatal */
                exit(2);
            }

            break;
        }

        ...

        /* for each listening socket */
        /* 為每個監聽套接字分配一個連接結構 */
        ls = cycle->listening.elts;
        for (i = 0; i < cycle->listening.nelts; i++) {

            c = ngx_get_connection(ls[i].fd, cycle->log);

            if (c == NULL) {
                return NGX_ERROR;
            }

            c->log = &ls[i].log;

            c->listening = &ls[i];
            ls[i].connection = c;

            rev = c->read;

            rev->log = c->log;
            /* 標識此讀事件為新請求連接事件 */
            rev->accept = 1;

            ...

    #if (NGX_WIN32)

            /* windows環境下不做分析，但原理類似 */

    #else
            /* 將讀事件結構的處理函數設置為ngx_event_accept */
            rev->handler = ngx_event_accept;
            /* 如果使用accept鎖的話，要在後面搶到鎖才能將監聽控制碼掛載上事件處理模型上 */
            if (ngx_use_accept_mutex) {
                continue;
            }
            /* 否則，將該監聽控制碼直接掛載上事件處理模型 */
            if (ngx_event_flags & NGX_USE_RTSIG_EVENT) {
                if (ngx_add_conn(c) == NGX_ERROR) {
                    return NGX_ERROR;
                }

            } else {
                if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                    return NGX_ERROR;
                }
            }

    #endif

        }

        return NGX_OK;
    }

當一個工作進程在某個時刻將監聽事件掛載上事件處理模型之後，nginx就可以正式的接收並處理用戶端過來的請求了。這時如果有一個用戶在流覽器的位址欄內輸入一個功能變數名稱，並且功能變數名稱解析伺服器將該功能變數名稱解析到一台由nginx監聽的伺服器上，nginx的事件處理模型接收到這個讀事件之後，會速度交由之前註冊好的事件處理函數ngx_event_accept來處理。

在ngx_event_accept函數中，nginx調用accept函數，從已連接佇列得到一個連接以及對應的套接字，接著分配一個連接結構（ngx_connection_t），並將新得到的套接字保存在該連接結構中，這裏還會做一些基本的連接初始化工作：

1, 首先給該連接分配一個記憶體池，初始大小默認為256位元組，可通過connection_pool_size指令設置；

2, 分配日誌結構，並保存在其中，以便後續的日誌系統使用；

3, 初始化連接相應的io收發函數，具體的io收發函數和使用的事件模型及作業系統相關；

4, 分配一個套介面位址（sockaddr），並將accept得到的對端位址拷貝在其中，保存在sockaddr欄位；

5, 將本地套介面位址保存在local_sockaddr欄位，因為這個值是從監聽結構ngx_listening_t中可得，而監聽結構中保存的只是配置檔中設置的監聽位址，但是配置的監聽位址可能是通配符*，即監聽在所有的位址上，所以連接中保存的這個值最終可能還會變動，會被確定為真正的接收位址；

6, 將連接的寫事件設置為已就緒，即設置ready為1，nginx默認連接第一次為可寫；

7, 如果監聽套接字設置了TCP_DEFER_ACCEPT屬性，則表示該連接上已經有資料包過來，於是設置讀事件為就緒；

8, 將sockaddr欄位保存的對端位址格式化為可讀字串，並保存在addr_text欄位；

最後調用ngx_http_init_connection函數初始化該連接結構的其他部分。

ngx_http_init_connection函數最重要的工作是初始化讀寫事件的處理函數：將該連接結構的寫事件的處理函數設置為ngx_http_empty_handler，這個事件處理函數不會做任何操作，實際上nginx默認連接第一次可寫，不會掛載寫事件，如果有資料需要發送，nginx會直接寫到這個連接，只有在發生一次寫不完的情況下，才會掛載寫事件到事件模型上，並設置真正的寫事件處理函數，這裏後面的章節還會做詳細介紹；讀事件的處理函數設置為ngx_http_init_request，此時如果該連接上已經有資料過來（設置了deferred accept)，則會直接調用ngx_http_init_request函數來處理該請求，反之則設置一個計時器並在事件處理模型上掛載一個讀事件，等待資料到來或者超時。當然這裏不管是已經有資料到來，或者需要等待資料到來，又或者等待超時，最終都會進入讀事件的處理函數-ngx_http_init_request。

ngx_http_init_request函數主要工作即是初始化請求，由於它是一個事件處理函數，它只有唯一一個ngx_event_t \*類型的參數，ngx_event_t 結構在nginx中表示一個事件，事件處理的上下文類似於一個中斷處理的上下文，為了在這個上下文得到相關的資訊，nginx中一般會將連接結構的引用保存在事件結構的data欄位，請求結構的引用則保存在連接結構的data欄位，這樣在事件處理函數中可以方便的得到對應的連接結構和請求結構。進入函數內部看一下，首先判斷該事件是否是超時事件，如果是的話直接關閉連接並返回；反之則是指之前accept的連接上有請求過來需要處理。

ngx_http_init_request函數首先在連接的記憶體池中為該請求分配一個ngx_http_request_t結構，這個結構將用來保存該請求所有的資訊。分配完之後，這個結構的引用會被包存在連接的hc成員的request欄位，以便於在長連接或pipelined請求中複用該請求結構。在這個函數中，nginx根據該請求的接收埠和位址找到一個默認虛擬伺服器配置（listen指令的default_server屬性用來標識一個默認虛擬伺服器，否則監聽在相同埠和位址的多個虛擬伺服器，其中第一個定義的則為默認）。

nginx配置檔中可以設置多個監聽在不同埠和位址的虛擬伺服器（每個server塊對應一個虛擬伺服器），另外還根據功能變數名稱（server_name指令可以配置該虛擬伺服器對應的功能變數名稱）來區分監聽在相同埠和位址的虛擬伺服器，每個虛擬伺服器可以擁有不同的配置內容，而這些配置內容決定了nginx在接收到一個請求之後如何處理該請求。找到之後，相應的配置被保存在該請求對應的ngx_http_request_t結構中。注意這裏根據埠和位址找到的默認配置只是臨時使用一下，最終nginx會根據功能變數名稱找到真正的虛擬伺服器配置，隨後的初始化工作還包括：

1, 將連接的讀事件的處理函數設置為ngx_http_process_request_line函數，這個函數用來解析請求行，將請求的read_event_handler設置為ngx_http_block_reading函數，這個函數實際上什麼都不做（當然在事件模型設置為水平觸發時，唯一做的事情就是將事件從事件模型監聽列表中刪除，防止該事件一直被觸發），後面會說到這裏為什麼會將read_event_handler設置為此函數；

2, 為這個請求分配一個緩衝區用來保存它的請求頭，位址保存在header_in欄位，默認大小為1024個位元組，可以使用client_header_buffer_size指令修改，這裏需要注意一下，nginx用來保存請求頭的緩衝區是在該請求所在連接的記憶體池中分配，而且會將位址保存一份在連接的buffer欄位中，這樣做的目的也是為了給該連接的下一次請求重用這個緩衝區，另外如果用戶端發過來的請求頭大於1024個位元組，nginx會重新分配更大的緩存區，默認用於大請求的頭的緩衝區最大為8K，最多4個，這2個值可以用large_client_header_buffers指令設置，後面還會說到請求行和一個請求頭都不能超過一個最大緩衝區的大小；

3, 為這個請求分配一個記憶體池，後續所有與該請求相關的記憶體分配一般都會使用該記憶體池，默認大小為4096個位元組，可以使用request_pool_size指令修改；

4, 為這個請求分配回應頭鏈表，初始大小為20；

5, 創建所有模組的上下文ctx指標陣列，變數資料；

6, 將該請求的main欄位設置為它本身，表示這是一個主請求，nginx中對應的還有子請求概念，後面的章節會做詳細的介紹；

7, 將該請求的count欄位設置為1，count欄位表示請求的引用計數；

8, 將當前時間保存在start_sec和start_msec欄位，這個時間是該請求的起始時刻，將被用來計算一個請求的處理時間（request time），nginx使用的這個起始點和apache略有差別，nginx中請求的起始點是接收到用戶端的第一個資料包開始，而apache則是接收到用戶端的整個request line後開始算起；

9, 初始化請求的其他欄位，比如將uri_changes設置為11，表示最多可以將該請求的uri改寫10次，subrequests被設置為201，表示一個請求最多可以發起200個子請求；

做完所有這些初始化工作之後，ngx_http_init_request函數會調用讀事件的處理函數來真正的解析用戶端發過來的資料，也就是會進入ngx_http_process_request_line函數中處理。

解析請求行 (99%)
+++++++++++++++++++++

ngx_http_process_request_line函數的主要作用即是解析請求行，同樣由於涉及到網路IO操作，即使是很短的一行請求行可能也不能被一次讀完，所以在之前的ngx_http_init_request函數中，ngx_http_process_request_line函數被設置為讀事件的處理函數，它也只擁有一個唯一的ngx_event_t \*類型參數，並且在函數的開頭，同樣需要判斷是否是超時事件，如果是的話，則關閉這個請求和連接；否則開始正常的解析流程。先調用ngx_http_read_request_header函數讀取資料。

由於可能多次進入ngx_http_process_request_line函數，ngx_http_read_request_header函數首先檢查請求的header_in指向的緩衝區內是否有資料，有的話直接返回；否則從連接讀取資料並保存在請求的header_in指向的緩存區，而且只要緩衝區有空間的話，會一次盡可能多的讀數據，讀到多少返回多少；如果用戶端暫時沒有發任何資料過來，並返回NGX_AGAIN，返回之前會做2件事情：

1，設置一個計時器，時長默認為60s，可以通過指令client_header_timeout設置，如果定時事件到達之前沒有任何可讀事件，nginx將會關閉此請求；

2，調用ngx_handle_read_event函數處理一下讀事件-如果該連接尚未在事件處理模型上掛載讀事件，則將其掛載上；

如果用戶端提前關閉了連接或者讀取資料發生了其他錯誤，則給用戶端返回一個400錯誤（當然這裏並不保證用戶端能夠接收到回應資料，因為用戶端可能都已經關閉了連接），最後函數返回NGX_ERROR；

如果ngx_http_read_request_header函數正常的讀取到了資料，ngx_http_process_request_line函數將調用ngx_http_parse_request_line函數來解析，這個函數根據http協定規範中對請求行的定義實現了一個有限狀態機，經過這個狀態機，nginx會記錄請求行中的請求方法（Method），請求uri以及http協議版本在緩衝區中的起始位置，在解析過程中還會記錄一些其他有用的資訊，以便後面的處理過程中使用。如果解析請求行的過程中沒有產生任何問題，該函數會返回NGX_OK；如果請求行不滿足協定規範，該函數會立即終止解析過程，並返回相應錯誤號；如果緩衝區資料不夠，該函數返回NGX_AGAIN。

在整個解析http請求的狀態機中始終遵循著兩條重要的原則：減少記憶體拷貝和回溯。

記憶體拷貝是一個相對比較昂貴的操作，大量的記憶體拷貝會帶來較低的運行時效率。nginx在需要做記憶體拷貝的地方儘量只拷貝記憶體的起始和結束位址而不是記憶體本身，這樣做的話僅僅只需要兩個賦值操作而已，大大降低了開銷，當然這樣帶來的影響是後續的操作不能修改記憶體本身，如果修改的話，會影響到所有引用到該記憶體區間的地方，所以必須很小心的管理，必要的時候需要拷貝一份。

這裏不得不提到nginx中最能體現這一思想的資料結構，ngx_buf_t，它用來表示nginx中的緩存，在很多情況下，只需要將一塊記憶體的起始位址和結束位址分別保存在它的pos和last成員中，再將它的memory標誌置1，即可表示一塊不能修改的記憶體區間，在另外的需要一塊能夠修改的緩存的情形中，則必須分配一塊所需大小的記憶體並保存其起始位址，再將ngx_bug_t的temprary標誌置1，表示這是一塊能夠被修改的記憶體區域。

再回到ngx_http_process_request_line函數中，如果ngx_http_parse_request_line函數返回了錯誤，則直接給用戶端返回400錯誤；
如果返回NGX_AGAIN，則需要判斷一下是否是由於緩衝區空間不夠，還是已讀數據不夠。如果是緩衝區大小不夠了，nginx會調用ngx_http_alloc_large_header_buffer函數來分配另一塊大緩衝區，如果大緩衝區還不夠裝下整個請求行，nginx則會返回414錯誤給用戶端，否則分配了更大的緩衝區並拷貝之前的資料之後，繼續調用ngx_http_read_request_header函數讀取資料來進入請求行自動機處理，直到請求行解析結束；

如果返回了NGX_OK，則表示請求行被正確的解析出來了，這時先記錄好請求行的起始位址以及長度，並將請求uri的path和參數部分保存在請求結構的uri欄位，請求方法起始位置和長度保存在method_name欄位，http版本起始位置和長度記錄在http_protocol欄位。還要從uri中解析出參數以及請求資源的拓展名，分別保存在args和exten欄位。接下來將要解析請求頭，將在下一小節中接著介紹。

解析請求頭 (99%)
+++++++++++++++++++++++

在ngx_http_process_request_line函數中，解析完請求行之後，如果請求行的uri裏面包含了功能變數名稱部分，則將其保存在請求結構的headers_in成員的server欄位，headers_in用來保存所有請求頭，它的類型為ngx_http_headers_in_t：


.. code-block:: none

    typedef struct {
        ngx_list_t                        headers;

        ngx_table_elt_t                  *host;
        ngx_table_elt_t                  *connection;
        ngx_table_elt_t                  *if_modified_since;
        ngx_table_elt_t                  *if_unmodified_since;
        ngx_table_elt_t                  *user_agent;
        ngx_table_elt_t                  *referer;
        ngx_table_elt_t                  *content_length;
        ngx_table_elt_t                  *content_type;

        ngx_table_elt_t                  *range;
        ngx_table_elt_t                  *if_range;

        ngx_table_elt_t                  *transfer_encoding;
        ngx_table_elt_t                  *expect;

    #if (NGX_HTTP_GZIP)
        ngx_table_elt_t                  *accept_encoding;
        ngx_table_elt_t                  *via;
    #endif

        ngx_table_elt_t                  *authorization;

        ngx_table_elt_t                  *keep_alive;

    #if (NGX_HTTP_PROXY || NGX_HTTP_REALIP || NGX_HTTP_GEO)
        ngx_table_elt_t                  *x_forwarded_for;
    #endif

    #if (NGX_HTTP_REALIP)
        ngx_table_elt_t                  *x_real_ip;
    #endif

    #if (NGX_HTTP_HEADERS)
        ngx_table_elt_t                  *accept;
        ngx_table_elt_t                  *accept_language;
    #endif

    #if (NGX_HTTP_DAV)
        ngx_table_elt_t                  *depth;
        ngx_table_elt_t                  *destination;
        ngx_table_elt_t                  *overwrite;
        ngx_table_elt_t                  *date;
    #endif

        ngx_str_t                         user;
        ngx_str_t                         passwd;

        ngx_array_t                       cookies;

        ngx_str_t                         server;
        off_t                             content_length_n;
        time_t                            keep_alive_n;

        unsigned                          connection_type:2;
        unsigned                          msie:1;
        unsigned                          msie6:1;
        unsigned                          opera:1;
        unsigned                          gecko:1;
        unsigned                          chrome:1;
        unsigned                          safari:1;
        unsigned                          konqueror:1;
    } ngx_http_headers_in_t;

接著，該函數會檢查進來的請求是否使用的是http0.9，如果是的話則使用從請求行裏得到的功能變數名稱，調用ngx_http_find_virtual_server（）函數來查找用來處理該請求的虛擬伺服器配置，之前通過埠和位址找到的默認配置不再使用，找到相應的配置之後，則直接調用ngx_http_process_request（）函數處理該請求，因為http0.9是最原始的http協議，它裏面沒有定義任何請求頭，顯然就不需要讀取請求頭的操作。

.. code-block:: none

            if (r->host_start && r->host_end) {

                host = r->host_start;
                n = ngx_http_validate_host(r, &host,
                                           r->host_end - r->host_start, 0);

                if (n == 0) {
                    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                                  "client sent invalid host in request line");
                    ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
                    return;
                }

                if (n < 0) {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    return;
                }

                r->headers_in.server.len = n;
                r->headers_in.server.data = host;
            }

            if (r->http_version < NGX_HTTP_VERSION_10) {

                if (ngx_http_find_virtual_server(r, r->headers_in.server.data,
                                                 r->headers_in.server.len)
                    == NGX_ERROR)
                {
                    ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    return;
                }

                ngx_http_process_request(r);
                return;
            }

當然，如果是1.0或者更新的http協議，接下來要做的就是讀取請求頭了，首先nginx會為請求頭分配空間，ngx_http_headers_in_t結構的headers欄位為一個鏈表結構，它被用來保存所有請求頭，初始為它分配了20個節點，每個節點的類型為ngx_table_elt_t，保存請求頭的name/value值對，還可以看到ngx_http_headers_in_t結構有很多類型為ngx_table_elt_t*的指標成員，而且從它們的命名可以看出是一些常見的請求頭名字，nginx對這些常用的請求頭在ngx_http_headers_in_t結構裏面保存了一份引用，後續需要使用的話，可以直接通過這些成員得到，另外也事先為cookie頭分配了2個元素的陣列空間，做完這些記憶體準備工作之後，該請求對應的讀事件結構的處理函數被設置為ngx_http_process_request_headers，並隨後馬上調用了該函數。

.. code-block:: none

            if (ngx_list_init(&r->headers_in.headers, r->pool, 20,
                              sizeof(ngx_table_elt_t))
                != NGX_OK)
            {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }


            if (ngx_array_init(&r->headers_in.cookies, r->pool, 2,
                               sizeof(ngx_table_elt_t *))
                != NGX_OK)
            {
                ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            c->log->action = "reading client request headers";

            rev->handler = ngx_http_process_request_headers;
            ngx_http_process_request_headers(rev);

ngx_http_process_request_headers函數迴圈的讀取所有的請求頭，並保存和初始化和請求頭相關的結構，下面詳細分析一下該函數：

因為nginx對讀取請求頭有超時限制，ngx_http_process_request_headers函數作為讀事件處理函數，一併處理了超時事件，如果讀超時了，nginx直接給該請求返回408錯誤：

.. code-block:: none

   if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

讀取和解析請求頭的邏輯和處理請求行差不多，總的流程也是迴圈的調用ngx_http_read_request_header（）函數讀取資料，然後再調用一個解析函數來從讀取的資料中解析請求頭，直到解析完所有請求頭，或者發生解析錯誤為主。當然由於涉及到網路io，這個流程可能發生在多個io事件的上下文中。

接著來細看該函數，先調用了ngx_http_read_request_header（）函數讀取資料，如果當前連接並沒有資料過來，再直接返回，等待下一次讀事件到來，如果讀到了一些資料則調用ngx_http_parse_header_line（）函數來解析，同樣的該解析函數實現為一個有限狀態機，邏輯很簡單，只是根據http協定來解析請求頭，每次調用該函數最多解析出一個請求頭，該函數返回4種不同返回值，表示不同解析結果：

1，返回NGX_OK，表示解析出了一行請求頭，這時還要判斷解析出的請求頭名字裏面是否有非法字元，名字裏面合法的字元包括字母，數位和連字元（-），另外如果設置了underscores_in_headers指令為on，則下劃線也是合法字元，但是nginx默認下劃線不合法，當請求頭裏面包含了非法的字元，nginx默認只是忽略這一行請求頭；如果一切都正常，nginx會將該請求頭及請求頭名字的hash值保存在請求結構體的headers_in成員的headers鏈表,而且對於一些常見的請求頭，如Host，Connection，nginx採用了類似於配置指令的方式，事先給這些請求頭分配了一個處理函數，當解析出一個請求頭時，會檢查該請求頭是否有設置處理函數，有的話則調用之，nginx所有有處理函數的請求頭都記錄在ngx_http_headers_in全局陣列中：

.. code-block:: none

    typedef struct {
        ngx_str_t                         name;
        ngx_uint_t                        offset;
        ngx_http_header_handler_pt        handler;
    } ngx_http_header_t;

    ngx_http_header_t  ngx_http_headers_in[] = {
        { ngx_string("Host"), offsetof(ngx_http_headers_in_t, host),
                     ngx_http_process_host },

        { ngx_string("Connection"), offsetof(ngx_http_headers_in_t, connection),
                     ngx_http_process_connection },

        { ngx_string("If-Modified-Since"),
                     offsetof(ngx_http_headers_in_t, if_modified_since),
                     ngx_http_process_unique_header_line },

        { ngx_string("If-Unmodified-Since"),
                     offsetof(ngx_http_headers_in_t, if_unmodified_since),
                     ngx_http_process_unique_header_line },

        { ngx_string("User-Agent"), offsetof(ngx_http_headers_in_t, user_agent),
                     ngx_http_process_user_agent },

        { ngx_string("Referer"), offsetof(ngx_http_headers_in_t, referer),
                     ngx_http_process_header_line },

        { ngx_string("Content-Length"),
                     offsetof(ngx_http_headers_in_t, content_length),
                     ngx_http_process_unique_header_line },

        { ngx_string("Content-Type"),
                     offsetof(ngx_http_headers_in_t, content_type),
                     ngx_http_process_header_line },

        { ngx_string("Range"), offsetof(ngx_http_headers_in_t, range),
                     ngx_http_process_header_line },

        { ngx_string("If-Range"),
                     offsetof(ngx_http_headers_in_t, if_range),
                     ngx_http_process_unique_header_line },

        { ngx_string("Transfer-Encoding"),
                     offsetof(ngx_http_headers_in_t, transfer_encoding),
                     ngx_http_process_header_line },

        { ngx_string("Expect"),
                     offsetof(ngx_http_headers_in_t, expect),
                     ngx_http_process_unique_header_line },

    #if (NGX_HTTP_GZIP)
        { ngx_string("Accept-Encoding"),
                     offsetof(ngx_http_headers_in_t, accept_encoding),
                     ngx_http_process_header_line },

        { ngx_string("Via"), offsetof(ngx_http_headers_in_t, via),
                     ngx_http_process_header_line },
    #endif

        { ngx_string("Authorization"),
                     offsetof(ngx_http_headers_in_t, authorization),
                     ngx_http_process_unique_header_line },

        { ngx_string("Keep-Alive"), offsetof(ngx_http_headers_in_t, keep_alive),
                     ngx_http_process_header_line },

    #if (NGX_HTTP_PROXY || NGX_HTTP_REALIP || NGX_HTTP_GEO)
        { ngx_string("X-Forwarded-For"),
                     offsetof(ngx_http_headers_in_t, x_forwarded_for),
                     ngx_http_process_header_line },
    #endif

    #if (NGX_HTTP_REALIP)
        { ngx_string("X-Real-IP"),
                     offsetof(ngx_http_headers_in_t, x_real_ip),
                     ngx_http_process_header_line },
    #endif

    #if (NGX_HTTP_HEADERS)
        { ngx_string("Accept"), offsetof(ngx_http_headers_in_t, accept),
                     ngx_http_process_header_line },

        { ngx_string("Accept-Language"),
                     offsetof(ngx_http_headers_in_t, accept_language),
                     ngx_http_process_header_line },
    #endif

    #if (NGX_HTTP_DAV)
        { ngx_string("Depth"), offsetof(ngx_http_headers_in_t, depth),
                     ngx_http_process_header_line },

        { ngx_string("Destination"), offsetof(ngx_http_headers_in_t, destination),
                     ngx_http_process_header_line },

        { ngx_string("Overwrite"), offsetof(ngx_http_headers_in_t, overwrite),
                     ngx_http_process_header_line },

        { ngx_string("Date"), offsetof(ngx_http_headers_in_t, date),
                     ngx_http_process_header_line },
    #endif

        { ngx_string("Cookie"), 0, ngx_http_process_cookie },

        { ngx_null_string, 0, NULL }
    };

ngx_http_headers_in陣列當前包含了25個常用的請求頭，每個請求頭都設置了一個處理函數，其中一部分請求頭設置的是公共處理函數，這裏有2個公共處理函數，ngx_http_process_header_line和ngx_http_process_unique_header_line。
先來看一下處理函數的函數指標定義：

.. code-block:: none

    typedef ngx_int_t (*ngx_http_header_handler_pt)(ngx_http_request_t *r,
        ngx_table_elt_t *h, ngx_uint_t offset);

它有3個參數，r為對應的請求結構，h為指向該請求頭在headers_in.headers鏈表中對應節點的指標，offset為該請求頭對應欄位在ngx_http_headers_in_t結構中的偏移。

再來看ngx_http_process_header_line函數：

.. code-block:: none

    static ngx_int_t
    ngx_http_process_header_line(ngx_http_request_t *r, ngx_table_elt_t *h,
        ngx_uint_t offset)
    {
        ngx_table_elt_t  **ph;

        ph = (ngx_table_elt_t **) ((char *) &r->headers_in + offset);

        if (*ph == NULL) {
            *ph = h;
        }

        return NGX_OK;
    }

這個函數只是簡單將該請求頭在ngx_http_headers_in_t結構中保存一份引用。ngx_http_process_unique_header_line功能類似，不同點在於該函數會檢查這個請求頭是否是重複的，如果是的話，則給該請求返回400錯誤。

ngx_http_headers_in陣列中剩下的請求頭都有自己特殊的處理函數，這些特殊的函數根據對應的請求頭有一些特殊的處理，下面拿Host頭的處理函數ngx_http_process_host做一下介紹：

.. code-block:: none

    static ngx_int_t
    ngx_http_process_host(ngx_http_request_t *r, ngx_table_elt_t *h,
        ngx_uint_t offset)
    {
        u_char   *host;
        ssize_t   len;

        if (r->headers_in.host == NULL) {
            r->headers_in.host = h;
        }

        host = h->value.data;
        len = ngx_http_validate_host(r, &host, h->value.len, 0);

        if (len == 0) {
            ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                          "client sent invalid host header");
            ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
            return NGX_ERROR;
        }

        if (len < 0) {
            ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_ERROR;
        }

        if (r->headers_in.server.len) {
            return NGX_OK;
        }

        r->headers_in.server.len = len;
        r->headers_in.server.data = host;

        return NGX_OK;
    }

此函數的目的也是保存Host頭的快速引用，它會對Host頭的值做一些合法性檢查，並從中解析出功能變數名稱，保存在headers_in.server欄位，實際上前面在解析請求行時，headers_in.server可能已經被賦值為從請求行中解析出來的功能變數名稱，根據http協定的規範，如果請求行中的uri帶有功能變數名稱的話，則功能變數名稱以它為准，所以這裏需檢查一下headers_in.server是否為空，如果不為空則不需要再賦值。

其他請求頭的特殊處理函數，不再做介紹，大致都是根據該請求頭在http協議中規定的意義及其值設置請求的一些屬性，必備後續使用。

對一個合法的請求頭的處理大致為如上所述；

2，返回NGX_AGAIN，表示當前接收到的資料不夠，一行請求頭還未結束，需要繼續下一輪迴圈。在下一輪迴圈中，nginx首先檢查請求頭緩衝區header_in是否已滿，如夠滿了，則調用ngx_http_alloc_large_header_buffer（）函數分配更多緩衝區，下面分析一下ngx_http_alloc_large_header_buffer函數：

.. code-block:: none

    static ngx_int_t
    ngx_http_alloc_large_header_buffer(ngx_http_request_t *r,
        ngx_uint_t request_line)
    {
        u_char                    *old, *new;
        ngx_buf_t                 *b;
        ngx_http_connection_t     *hc;
        ngx_http_core_srv_conf_t  *cscf;

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http alloc large header buffer");

        /*
         * 在解析請求行階段，如果用戶端在發送請求行之前發送了大量回車換行符將
         * 緩衝區塞滿了，針對這種情況，nginx只是簡單的重置緩衝區，丟棄這些垃圾
         * 資料，不需要分配更大的記憶體。
         */
        if (request_line && r->state == 0) {

            /* the client fills up the buffer with "\r\n" */

            r->request_length += r->header_in->end - r->header_in->start;

            r->header_in->pos = r->header_in->start;
            r->header_in->last = r->header_in->start;

            return NGX_OK;
        }

        /* 保存請求行或者請求頭在舊緩衝區中的起始位址 */
        old = request_line ? r->request_start : r->header_name_start;

        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);

        /* 如果一個大緩衝區還裝不下請求行或者一個請求頭，則返回錯誤 */
        if (r->state != 0
            && (size_t) (r->header_in->pos - old)
                                         >= cscf->large_client_header_buffers.size)
        {
            return NGX_DECLINED;
        }

        hc = r->http_connection;

        /* 首先在ngx_http_connection_t結構中查找是否有空閒緩衝區，有的話，直接取之 */
        if (hc->nfree) {
            b = hc->free[--hc->nfree];

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http large header free: %p %uz",
                           b->pos, b->end - b->last);

        /* 檢查給該請求分配的請求頭緩衝區個數是否已經超過限制，默認最大個數為4個 */
        } else if (hc->nbusy < cscf->large_client_header_buffers.num) {

            if (hc->busy == NULL) {
                hc->busy = ngx_palloc(r->connection->pool,
                      cscf->large_client_header_buffers.num * sizeof(ngx_buf_t *));
                if (hc->busy == NULL) {
                    return NGX_ERROR;
                }
            }

            /* 如果還沒有達到最大分配數量，則分配一個新的大緩衝區 */
            b = ngx_create_temp_buf(r->connection->pool,
                                    cscf->large_client_header_buffers.size);
            if (b == NULL) {
                return NGX_ERROR;
            }

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http large header alloc: %p %uz",
                           b->pos, b->end - b->last);

        } else {
            /* 如果已經達到最大的分配限制，則返回錯誤 */
            return NGX_DECLINED;
        }

        /* 將從空閒佇列取得的或者新分配的緩衝區加入已使用佇列 */
        hc->busy[hc->nbusy++] = b;

        /*
         * 因為nginx中，所有的請求頭的保存形式都是指標（起始和結束位址），
         * 所以一行完整的請求頭必須放在連續的記憶體塊中。如果舊的緩衝區不能
         * 再放下整行請求頭，則分配新緩衝區，並從舊緩衝區拷貝已經讀取的部分請求頭，
         * 拷貝完之後，需要修改所有相關指標指向到新緩衝區。
         * status為0表示解析完一行請求頭之後，緩衝區正好被用完，這種情況不需要拷貝
         */
        if (r->state == 0) {
            /*
             * r->state == 0 means that a header line was parsed successfully
             * and we do not need to copy incomplete header line and
             * to relocate the parser header pointers
             */

            r->request_length += r->header_in->end - r->header_in->start;

            r->header_in = b;

            return NGX_OK;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http large header copy: %d", r->header_in->pos - old);

        r->request_length += old - r->header_in->start;

        new = b->start;

        /* 拷貝舊緩衝區中不完整的請求頭 */
        ngx_memcpy(new, old, r->header_in->pos - old);

        b->pos = new + (r->header_in->pos - old);
        b->last = new + (r->header_in->pos - old);

        /* 修改相應的指標指向新緩衝區 */
        if (request_line) {
            r->request_start = new;

            if (r->request_end) {
                r->request_end = new + (r->request_end - old);
            }

            r->method_end = new + (r->method_end - old);

            r->uri_start = new + (r->uri_start - old);
            r->uri_end = new + (r->uri_end - old);

            if (r->schema_start) {
                r->schema_start = new + (r->schema_start - old);
                r->schema_end = new + (r->schema_end - old);
            }

            if (r->host_start) {
                r->host_start = new + (r->host_start - old);
                if (r->host_end) {
                    r->host_end = new + (r->host_end - old);
                }
            }

            if (r->port_start) {
                r->port_start = new + (r->port_start - old);
                r->port_end = new + (r->port_end - old);
            }

            if (r->uri_ext) {
                r->uri_ext = new + (r->uri_ext - old);
            }

            if (r->args_start) {
                r->args_start = new + (r->args_start - old);
            }

            if (r->http_protocol.data) {
                r->http_protocol.data = new + (r->http_protocol.data - old);
            }

        } else {
            r->header_name_start = new;
            r->header_name_end = new + (r->header_name_end - old);
            r->header_start = new + (r->header_start - old);
            r->header_end = new + (r->header_end - old);
        }

        r->header_in = b;

        return NGX_OK;
    }

當ngx_http_alloc_large_header_buffer函數返回NGX_DECLINED時，表示用戶端發送了一行過大的請求頭，或者是整個請求頭部超過了限制，nginx會返回494錯誤，注意到nginx再返回494錯誤之前將請求的lingering_close標識置為了1，這樣做的目的是在返回回應之丟棄掉用戶端發過來的其他資料；

3，返回NGX_HTTP_PARSE_INVALID_HEADER，表示請求頭解析過程中遇到錯誤，一般為用戶端發送了不符合協定規範的頭部，此時nginx返回400錯誤；

4，返回NGX_HTTP_PARSE_HEADER_DONE，表示所有請求頭已經成功的解析，這時請求的狀態被設置為NGX_HTTP_PROCESS_REQUEST_STATE，意味著結束了請求讀取階段，正式進入了請求處理階段，但是實際上請求可能含有請求體，nginx在請求讀取階段並不會去讀取請求體，這個工作交給了後續的請求處理階段的模組，這樣做的目的是nginx本身並不知道這些請求體是否有用，如果後續模組並不需要的話，一方面請求體一般較大，如果全部讀取進記憶體，則白白耗費大量的記憶體空間，另一方面即使nginx將請求體寫進磁片，但是涉及到磁片io，會耗費比較多時間。所以交由後續模組來決定讀取還是丟棄請求體是最明智的辦法。

讀取完請求頭之後，nginx調用了ngx_http_process_request_header（）函數，這個函數主要做了兩個方面的事情，一是調用ngx_http_find_virtual_server（）函數查找虛擬伺服器配置；二是對一些請求頭做一些協議的檢查。比如對那些使用http1.1協議但是卻沒有發送Host頭的請求，nginx給這些請求返回400錯誤。還有nginx現在的版本並不支持chunked格式的輸入，如果某些請求申明自己使用了chunked格式的輸入（請求帶有值為chunked的transfer_encoding頭部)，nginx給這些請求返回411錯誤。等等。

最後調用ngx_http_process_request（）函數處理請求,至此，nginx請求頭接收流程就介紹完畢。



請求體讀取(100%)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

上節說到nginx核心本身不會主動讀取請求體，這個工作是交給請求處理階段的模組來做，但是nginx核心提供了ngx_http_read_client_request_body()介面來讀取請求體，另外還提供了一個丟棄請求體的介面-ngx_http_discard_request_body()，在請求執行的各個階段中，任何一個階段的模組如果對請求體感興趣或者希望丟掉用戶端發過來的請求體，可以分別調用這兩個介面來完成。這兩個介面是nginx核心提供的處理請求體的標準介面，如果希望配置檔中一些請求體相關的指令（比如client_body_in_file_only，client_body_buffer_size等）能夠預期工作，以及能夠正常使用nginx內置的一些和請求體相關的變數（比如$request_body和$request_body_file），一般來說所有模組都必須調用這些介面來完成相應操作，如果需要自定義介面來處理請求體，也應儘量相容nginx默認的行為。

讀取請求體
+++++++++++++

請求體的讀取一般發生在nginx的content handler中，一些nginx內置的模組，比如proxy模組，fastcgi模組，uwsgi模組等，這些模組的行為必須將用戶端過來的請求體（如果有的話）以相應協議完整的轉發到後端服務進程，所有的這些模組都是調用了ngx_http_read_client_request_body()介面來完成請求體讀取。值得注意的是這些模組會把用戶端的請求體完整的讀取後才開始往後端轉發資料。

由於記憶體的限制，ngx_http_read_client_request_body()介面讀取的請求體會部分或者全部寫入一個暫存檔案中，根據請求體的大小以及相關的指令配置，請求體可能完整放置在一塊連續記憶體中，也可能分別放置在兩塊不同記憶體中，還可能全部存在一個暫存檔案中，最後還可能一部分在記憶體，剩餘部分在暫存檔案中。下面先介紹一下和這些不同存儲行為相關的指令\：

:client_body_buffer_size: 設置緩存請求體的buffer大小，默認為系統頁大小的2倍，當請求體的大小超過此大小時，nginx會把請求體寫入到暫存檔案中。可以根據業務需求設置合適的大小，儘量避免磁片io操作;

:client_body_in_single_buffer: 指示是否將請求體完整的存儲在一塊連續的記憶體中，默認為off，如果此指令被設置為on，則nginx會保證請求體在不大於client_body_buffer_size設置的值時，被存放在一塊連續的記憶體中，但超過大小時會被整個寫入一個暫存檔案;

:client_body_in_file_only: 設置是否總是將請求體保存在暫存檔案中，默認為off，當此指定被設置為on時，即使用戶端顯示指示了請求體長度為0時，nginx還是會為請求創建一個暫存檔案。

接著介紹ngx_http_read_client_request_body()介面的實現，它的定義如下：

.. code-block:: none

    ngx_int_t
    ngx_http_read_client_request_body(ngx_http_request_t *r,
        ngx_http_client_body_handler_pt post_handler)

該介面有2個參數，第1個為指向請求結構的指標，第2個為一個函數指標，當請求體讀完時，它會被調用。之前也說到根據nginx現有行為，模組邏輯會在請求體讀完後執行，這個回調函數一般就是模組的邏輯處理函數。ngx_http_read_client_request_body()函數首先將參數r對應的主請求的引用加1，這樣做的目的和該介面被調用的上下文有關，一般而言，模組是在content handler中調用此介面，一個典型的調用如下：

.. code-block:: none

    static ngx_int_t
    ngx_http_proxy_handler(ngx_http_request_t *r)
    {
        ...
        rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);


        if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
            return rc;
        }

        return NGX_DONE;
    }

上面的代碼是在porxy模組的content handler，ngx_http_proxy_handler()中調用了ngx_http_read_client_request_body()函數，其中ngx_http_upstream_init()被作為回調函數傳入進介面中，另外nginx中模組的content handler調用的上下文如下：

.. code-block:: none

    ngx_int_t
    ngx_http_core_content_phase(ngx_http_request_t *r,
        ngx_http_phase_handler_t *ph)
    {
        ...
        if (r->content_handler) {
            r->write_event_handler = ngx_http_request_empty_handler;
            ngx_http_finalize_request(r, r->content_handler(r));
            return NGX_OK;
        }
        ...
    }

上面的代碼中，content handler調用之後，它的返回值作為參數調用了ngx_http_finalize_request()函數，在請求體沒有被接收完全時，ngx_http_read_client_request_body()函數返回值為NGX_AGAIN，此時content handler，比如ngx_http_proxy_handler()會返回NGX_DONE，而NGX_DONE作為參數傳給ngx_http_finalize_request()函數會導致主請求的引用計數減1，所以正好抵消了ngx_http_read_client_request_body()函數開頭對主請求計數的加1。

接下來回到ngx_http_read_client_request_body()函數，它會檢查該請求的請求體是否已經被讀取或者被丟棄了，如果是的話，則直接調用回調函數並返回NGX_OK，這裏實際上是為子請求檢查，子請求是nginx中的一個概念，nginx中可以在當前請求中發起另外一個或多個全新的子請求來訪問其他的location，關於子請求的具體介紹會在後面的章節作詳細分析，一般而言子請求不需要自己去讀取請求體。

函數接著調用ngx_http_test_expect()檢查用戶端是否發送了Expect: 100-continue頭，是的話則給用戶端回復"HTTP/1.1 100 Continue"，根據http 1.1協定，用戶端可以發送一個Expect頭來向伺服器表明期望發送請求體，伺服器如果允許用戶端發送請求體，則會回復"HTTP/1.1 100 Continue"，用戶端收到時，才會開始發送請求體。

接著繼續為接收請求體做準備工作，分配一個ngx_http_request_body_t結構，並保存在r->request_body，這個結構用來保存請求體讀取過程用到的緩存引用，暫存檔案引用，剩餘請求體大小等資訊，它的定義如下:

.. code-block:: none

    typedef struct {
        ngx_temp_file_t                  *temp_file;
        ngx_chain_t                      *bufs;
        ngx_buf_t                        *buf;
        off_t                             rest;
        ngx_chain_t                      *to_write;
        ngx_http_client_body_handler_pt   post_handler;
    } ngx_http_request_body_t;

:temp_file: 指向儲存請求體的暫存檔案的指標；

:bufs: 指向保存請求體的鏈表頭；

:buf: 指向當前用於保存請求體的記憶體緩存；

:rest: 當前剩餘的請求體大小；

:post_handler: 保存傳給ngx_http_read_client_request_body()函數的回調函數。

做好準備工作之後，函數開始檢查請求是否帶有content_length頭，如果沒有該頭或者用戶端發送了一個值為0的content_length頭，表明沒有請求體，這時直接調用回調函數並返回NGX_OK即可。當然如果client_body_in_file_only指令被設置為on，且content_length為0時，該函數在調用回調函數之前，會創建一個空的暫存檔案。

進入到函數下半部分，表明用戶端請求確實表明了要發送請求體，該函數會先檢查是否在讀取請求頭時預讀了請求體，這裏的檢查是通過判斷保存請求頭的緩存(r->header_in)中是否還有未處理的資料。如果有預讀數據，則分配一個ngx_buf_t結構，並將r->header_in中的預讀數據保存在其中，並且如果r->header_in中還有剩餘空間，並且能夠容下剩餘未讀取的請求體，這些空間將被繼續使用，而不用分配新的緩存，當然甚至如果請求體已經被整個預讀了，則不需要繼續處理了，此時調用回調函數後返回。

如果沒有預讀數據或者預讀不完整，該函數會分配一塊新的記憶體（除非r->header_in還有足夠的剩餘空間），另外如果request_body_in_single_buf指令被設置為no，則預讀的資料會被拷貝進新開闢的記憶體塊中，真正讀取請求體的操作是在ngx_http_do_read_client_request_body()函數，該函數迴圈的讀取請求體並保存在緩存中，如果緩存被寫滿了，其中的資料會被清空並寫回到暫存檔案中。當然這裏有可能不能一次將資料讀到，該函數會掛載讀事件並設置讀事件handler為ngx_http_read_client_request_body_handler，另外nginx核心對兩次請求體的讀事件之間也做了超時設置，client_body_timeout指令可以設置這個超時時間，默認為60秒，如果下次讀事件超時了，nginx會返回408給用戶端。

最終讀完請求體後，ngx_http_do_read_client_request_body()會根據配置，將請求體調整到預期的位置(記憶體或者檔)，所有情況下請求體都可以從r->request_body的bufs鏈表得到，該鏈表最多可能有2個節點，每個節點為一個buffer，但是這個buffer的內容可能是保存在記憶體中，也可能是保存在磁片檔中。另外$request_body變數只在當請求體已經被讀取並且是全部保存在記憶體中，才能取得相應的資料。

丟棄請求體
+++++++++++++

一個模組想要主動的丟棄用戶端發過的請求體，可以調用nginx核心提供的ngx_http_discard_request_body()介面，主動丟棄的原因可能有很多種，如模組的業務邏輯壓根不需要請求體 ，用戶端發送了過大的請求體，另外為了相容http1.1協定的pipeline請求，模組有義務主動丟棄不需要的請求體。總之為了保持良好的用戶端相容性，nginx必須主動丟棄無用的請求體。下面開始分析ngx_http_discard_request_body()函數：

.. code-block:: none

    ngx_int_t
    ngx_http_discard_request_body(ngx_http_request_t *r)
    {
        ssize_t       size;
        ngx_event_t  *rev;

        if (r != r->main || r->discard_body) {
            return NGX_OK;
        }

        if (ngx_http_test_expect(r) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        rev = r->connection->read;

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0, "http set discard body");

        if (rev->timer_set) {
            ngx_del_timer(rev);
        }

        if (r->headers_in.content_length_n <= 0 || r->request_body) {
            return NGX_OK;
        }

        size = r->header_in->last - r->header_in->pos;

        if (size) {
            if (r->headers_in.content_length_n > size) {
                r->header_in->pos += size;
                r->headers_in.content_length_n -= size;

            } else {
                r->header_in->pos += (size_t) r->headers_in.content_length_n;
                r->headers_in.content_length_n = 0;
                return NGX_OK;
            }
        }

        r->read_event_handler = ngx_http_discarded_request_body_handler;

        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        if (ngx_http_read_discarded_request_body(r) == NGX_OK) {
            r->lingering_close = 0;

        } else {
            r->count++;
            r->discard_body = 1;
        }

        return NGX_OK;
    }

由於函數不長，這裏把它完整的列出來了，函數的開始同樣先判斷了不需要再做處理的情況：子請求不需要處理，已經調用過此函數的也不需要再處理。接著調用ngx_http_test_expect() 處理http1.1 expect的情況，根據http1.1的expect機制，如果用戶端發送了expect頭，而服務端不希望接收請求體時，必須返回417(Expectation Failed)錯誤。nginx並沒有這樣做，它只是簡單的讓用戶端把請求體發送過來，然後丟棄掉。接下來，函數刪掉了讀事件上的計時器，因為這時本身就不需要請求體，所以也無所謂用戶端發送的快還是慢了，當然後面還會講到，當nginx已經處理完該請求但用戶端還沒有發送完無用的請求體時，nginx會在讀事件上再掛上計時器。

用戶端如果打算發送請求體，就必須發送content-length頭，所以函數會檢查請求頭中的content-length頭，同時還會查看其他地方是不是已經讀取了請求體。如果確實有待處理的請求體，函數接著檢查請求頭buffer中預讀的資料，預讀的資料會直接被丟掉，當然如果請求體已經被全部預讀，函數就直接返回了。

接下來，如果還有剩餘的請求體未處理，該函數調用ngx_handle_read_event()在事件處理機制中掛載好讀事件，並把讀事件的處理函數設置為ngx_http_discarded_request_body_handler。做好這些準備之後，該函數最後調用ngx_http_read_discarded_request_body()介面讀取用戶端過來的請求體並丟棄。如果用戶端並沒有一次將請求體發過來，函數會返回，剩餘的資料等到下一次讀事件過來時，交給ngx_http_discarded_request_body_handler()來處理，這時，請求的discard_body將被設置為1用來標識這種情況。另外請求的引用數(count)也被加1，這樣做的目的是用戶端可能在nginx處理完請求之後仍未完整發送待發送的請求體，增加引用是防止nginx核心在處理完請求後直接釋放了請求的相關資源。

ngx_http_read_discarded_request_body()函數非常簡單，它迴圈的從鏈結中讀取資料並丟棄，直到讀完接收緩衝區的所有資料，如果請求體已經被讀完了，該函數會設置讀事件的處理函數為ngx_http_block_reading，這個函數僅僅刪除水平觸發的讀事件，防止同一事件不斷被觸發。

最後看一下讀事件的處理函數ngx_http_discarded_request_body_handler，這個函數每次讀事件來時會被調用，先看一下它的源碼：

.. code-block:: none

    void
    ngx_http_discarded_request_body_handler(ngx_http_request_t *r)
    {
        ...

        c = r->connection;
        rev = c->read;

        if (rev->timedout) {
            c->timedout = 1;
            c->error = 1;
            ngx_http_finalize_request(r, NGX_ERROR);
            return;
        }

        if (r->lingering_time) {
            timer = (ngx_msec_t) (r->lingering_time - ngx_time());

            if (timer <= 0) {
                r->discard_body = 0;
                r->lingering_close = 0;
                ngx_http_finalize_request(r, NGX_ERROR);
                return;
            }

        } else {
            timer = 0;
        }

        rc = ngx_http_read_discarded_request_body(r);

        if (rc == NGX_OK) {
            r->discard_body = 0;
            r->lingering_close = 0;
            ngx_http_finalize_request(r, NGX_DONE);
            return;
        }

        /* rc == NGX_AGAIN */

        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            c->error = 1;
            ngx_http_finalize_request(r, NGX_ERROR);
            return;
        }

        if (timer) {

            clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

            timer *= 1000;

            if (timer > clcf->lingering_timeout) {
                timer = clcf->lingering_timeout;
            }

            ngx_add_timer(rev, timer);
        }
    }

函數一開始就處理了讀事件超時的情況，之前說到在ngx_http_discard_request_body()函數中已經刪除了讀事件的計時器，那麼什麼時候會設置計時器呢？答案就是在nginx已經處理完該請求，但是又沒有完全將該請求的請求體丟棄的時候（用戶端可能還沒有發送過來），在ngx_http_finalize_connection()函數中，如果檢查到還有未丟棄的請求體時，nginx會添加一個讀事件計時器，它的時長為lingering_timeout指令所指定，默認為5秒，不過這個時間僅僅兩次讀事件之間的超時時間，等待請求體的總時長為lingering_time指令所指定，默認為30秒。這種情況中，該函數如果檢測到超時事件則直接返回並斷開連接。同樣，還需要控制整個丟棄請求體的時長不能超過lingering_time設置的時間，如果超過了最大時長，也會直接返回並斷開連接。

如果讀事件發生在請求處理完之前，則不用處理超時事件，也不用設置計時器，函數只是簡單的調用ngx_http_read_discarded_request_body()來讀取並丟棄資料。


多階段處理請求
--------------------------



find-config階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



rewrite階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



post-rewrite階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



access階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



post-access階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



content階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



log階段
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



返回回應資料
-----------------------



header filter分析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



body filter分析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



finalize_request函數分析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



特殊回應
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



chunked回應體
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



pipeline請求
-------------------



keepalive請求
--------------------



subrequest原理解析 (99%)
-----------------------------

子請求並不是http標準裏面的概念，它是在當前請求中發起的一個新的請求，它擁有自己的ngx_http_request_t結構，uri和args。一般來說使用subrequest的效率可能會有些影響，因為它需要重新從server rewrite開始走一遍request處理的PHASE，但是它在某些情況下使用能帶來方便，比較常用的是用subrequest來訪問一個upstream的後端，並給它一個ngx_http_post_subrequest_t的回調handler，這樣有點類似於一個非同步的函數調用。對於從upstream返回的資料，subrequest允許根據創建時指定的flag，來決定由用戶自己處理(回調handler中)還是由upstream模組直接發送到out put filter。簡單的說一下subrequest的行為，nginx使用subrequest訪問某個location，產生相應的資料，並插入到nginx輸出鏈的相應位置（創建subrequest時的位置），下面用nginx代碼內的addition模組(默認未編譯進nginx核心，請使用--with-http_addition_module選項包含此模組)來舉例說明一下：

.. code-block:: none

    location /main.htm {
        # content of main.htm: main
        add_before_body /hello.htm;
        add_after_body /world.htm;
    }
    location /hello.htm {
        #content of hello.htm: hello
    }
    location /world.htm {
        #content of world.htm: world
    }
訪問/main.htm，將得到如下回應：

.. code-block:: none

    hello
    main
    world

上面的add_before_body指令發起一個subrequest來訪問/hello.htm，並將產生的內容(hello)插入主請求響應體的開頭，add_after_body指令發起一個subrequest訪問/world.htm，並將產生的內容(world)附加在主請求回應體的結尾。addition模組是一個filter模組，但是subrequest既可以在phase模組中使用，也可以在filter模組中使用。

在進行源碼解析之前，先來想想如果是我們自己要實現subrequest的上述行為，該如何來做？subrequest還可能有自己的subrequest，而且每個subrequest都不一定按照其創建的順序來輸出資料，所以簡單的採用鏈表不好實現，於是進一步聯想到可以採用樹的結構來做，主請求即為根節點，每個節點可以有自己的子節點，遍曆某節點表示處理某請求，自然的可以想到這裏可能是用後根(序)遍曆的方法，沒錯，實際上Igor採用樹和鏈表結合的方式實現了subrequest的功能，但是由於節點（請求）產生資料的順序不是固定按節點創建順序(左->右)，而且可能分多次產生資料，不能簡單的用後根(序)遍曆。Igor使用了2個鏈表的結構來實現，第一個是每個請求都有的postponed鏈表，一般情況下每個鏈表節點保存了該請求的一個子請求，該鏈表節點定義如下：

.. code-block:: none

    struct ngx_http_postponed_request_s {
        ngx_http_request_t               *request;
        ngx_chain_t                      *out;
        ngx_http_postponed_request_t     *next;
    };

可以看到它有一個request欄位，可以用來保存子請求，另外還有一個ngx_chain_t類型的out欄位，實際上一個請求的postponed鏈表裏面除了保存子請求的節點，還有保存該請求自己產生的資料的節點，資料保存在out欄位；第二個是posted_requests鏈表，它掛載了當前需要遍曆的請求（節點）， 該鏈表保存在主請求（根節點）的posted_requests欄位，鏈表節點定義如下：

.. code-block:: none

    struct ngx_http_posted_request_s {
        ngx_http_request_t               *request;
        ngx_http_posted_request_t        *next;
    };

在ngx_http_run_posted_requests函數中會順序的遍曆主請求的posted_requests鏈表：

.. code-block:: none

    void
    ngx_http_run_posted_requests(ngx_connection_t *c)
    {
        ...
        for ( ;; ) {
            /* 連接已經斷開，直接返回 */
            if (c->destroyed) {
                return;
            }

            r = c->data;
            /* 從posted_requests鏈表的隊頭開始遍曆 */
            pr = r->main->posted_requests;

            if (pr == NULL) {
                return;
            }
          

            /* 從鏈表中移除即將要遍曆的節點 */
            r->main->posted_requests = pr->next;
            /* 得到該節點中保存的請求 */
            r = pr->request;

            ctx = c->log->data;
            ctx->current_request = r;

            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                           "http posted request: \"%V?%V\"", &r->uri, &r->args);
            /* 遍曆該節點（請求） */
            r->write_event_handler(r);
        }
    }

ngx_http_run_posted_requests函數的調用點後面會做說明。

瞭解了一些實現的原理，來看代碼就簡單多了，現在正式進行subrequest的源碼解析， 首先來看一下創建subrequest的函數定義：

.. code-block:: none

    ngx_int_t
    ngx_http_subrequest(ngx_http_request_t *r,
        ngx_str_t *uri, ngx_str_t *args, ngx_http_request_t **psr,
        ngx_http_post_subrequest_t *ps, ngx_uint_t flags)

參數r為當前的請求，uri和args為新的要發起的uri和args，當然args可以為NULL，psr為指向一個ngx_http_request_t指標的指標，它的作用就是獲得創建的子請求，ps的類型為ngx_http_post_subrequest_t，它的定義如下：

.. code-block:: none

    typedef struct {
        ngx_http_post_subrequest_pt       handler;
        void                             *data;
    } ngx_http_post_subrequest_t;

    typedef ngx_int_t (*ngx_http_post_subrequest_pt)(ngx_http_request_t *r,
        void *data, ngx_int_t rc);

它就是之前說到的回調handler，結構裏面的handler類型為ngx_http_post_subrequest_pt，它是函數指標，data為傳遞給handler的額外參數。再來看一下ngx_http_subrequest函數的最後一個是flags，現在的源碼中實際上只有2種類型的flag，分別為NGX_HTTP_SUBREQUEST_IN_MEMORY和NGX_HTTP_SUBREQUEST_WAITED，第一個就是指定文章開頭說到的子請求的upstream處理資料的方式，第二個參數表示如果該子請求提前完成(按後續遍曆的順序)，是否設置將它的狀態設為done，當設置該參數時，提前完成就會設置done，不設時，會讓該子請求等待它之前的子請求處理完畢才會將狀態設置為done。

進入ngx_http_subrequest函數內部看看：

.. code-block:: none

    {
        ...
        /* 解析flags， subrequest_in_memory在upstream模組解析完頭部，
           發送body給downsstream時用到 */
        sr->subrequest_in_memory = (flags & NGX_HTTP_SUBREQUEST_IN_MEMORY) != 0;
        sr->waited = (flags & NGX_HTTP_SUBREQUEST_WAITED) != 0;

        sr->unparsed_uri = r->unparsed_uri;
        sr->method_name = ngx_http_core_get_method;
        sr->http_protocol = r->http_protocol;

        ngx_http_set_exten(sr);
        /* 主請求保存在main欄位中 */
        sr->main = r->main;
        /* 父請求為當前請求 */   
        sr->parent = r;
        /* 保存回調handler及資料，在子請求執行完，將會調用 */
        sr->post_subrequest = ps;
        /* 讀事件handler賦值為不做任何事的函數，因為子請求不用再讀數據或者檢查連接狀態；
           寫事件handler為ngx_http_handler，它會重走phase */
        sr->read_event_handler = ngx_http_request_empty_handler;
        sr->write_event_handler = ngx_http_handler;

        /* ngx_connection_s的data欄位比較關鍵，它保存了當前可以向out chain輸出資料的請求，
           具體意義後面會做詳細介紹 */
        if (c->data == r && r->postponed == NULL) {
            c->data = sr;
        }
        /* 默認共用父請求的變數，當然你也可以根據需求在創建完子請求後，再創建子請求獨立的變數集 */
        sr->variables = r->variables;

        sr->log_handler = r->log_handler;

        pr = ngx_palloc(r->pool, sizeof(ngx_http_postponed_request_t));
        if (pr == NULL) {
            return NGX_ERROR;
        }

        pr->request = sr;
        pr->out = NULL;
        pr->next = NULL;
        /* 把該子請求掛載在其父請求的postponed鏈表的隊尾 */
        if (r->postponed) {
            for (p = r->postponed; p->next; p = p->next) { /* void */ }
            p->next = pr;

        } else {
            r->postponed = pr;
        }
        /* 子請求為內部請求，它可以訪問internal類型的location */
        sr->internal = 1;
        /* 繼承父請求的一些狀態 */
        sr->discard_body = r->discard_body;
        sr->expect_tested = 1;
        sr->main_filter_need_in_memory = r->main_filter_need_in_memory;

        sr->uri_changes = NGX_HTTP_MAX_URI_CHANGES + 1;

        tp = ngx_timeofday();
        r->start_sec = tp->sec;
        r->start_msec = tp->msec;

        r->main->subrequests++;
        /* 增加主請求的引用數，這個欄位主要是在ngx_http_finalize_request調用的一些結束請求和
           連接的函數中使用 */
        r->main->count++;

        *psr = sr;
        /* 將該子請求掛載在主請求的posted_requests鏈表隊尾 */
        return ngx_http_post_request(sr, NULL);
    }

到這時，子請求創建完畢，一般來說子請求的創建都發生在某個請求的content handler或者某個filter內，從上面的函數可以看到子請求並沒有馬上被執行，只是被掛載在了主請求的posted_requests鏈表中，那它什麼時候可以執行呢？之前說到posted_requests鏈表是在ngx_http_run_posted_requests函數中遍曆，那麼ngx_http_run_posted_requests函數又是在什麼時候調用？它實際上是在某個請求的讀（寫）事件的handler中，執行完該請求相關的處理後被調用，比如主請求在走完一遍PHASE的時候會調用ngx_http_run_posted_requests，這時子請求得以運行。

這時實際還有1個問題需要解決，由於nginx是多進程，是不能夠隨意阻塞的（如果一個請求阻塞了當前進程，就相當於阻塞了這個進程accept到的所有其他請求，同時該進程也不能accept新請求），一個請求可能由於某些原因需要阻塞（比如訪問io），nginx的做法是設置該請求的一些狀態並在epoll中添加相應的事件，然後轉去處理其他請求，等到該事件到來時再繼續處理該請求，這樣的行為就意味著一個請求可能需要多次執行機會才能完成，對於一個請求的多個子請求來說，意味著它們完成的先後順序可能和它們創建的順序是不一樣的，所以必須有一種機制讓提前完成的子請求保存它產生的資料，而不是直接輸出到out chain，同時也能夠讓當前能夠往out chain輸出資料的請求及時的輸出產生的資料。作者Igor採用ngx_connection_t中的data欄位，以及一個body filter，即ngx_http_postpone_filter，還有ngx_http_finalize_request函數中的一些邏輯來解決這個問題。

下面用一個圖來做說明，下圖是某時刻某個主請求和它的所有子孫請求的樹結構：

.. image:: /images/chapter-12-1.png
    :height:  273 px
    :width:   771 px
    :scale:   80 %
    :align:   center

圖中的root節點即為主請求，它的postponed鏈表從左至右掛載了3個節點，SUB1是它的第一個子請求，DATA1是它產生的一段資料，SUB2是它的第2個子請求，而且這2個子請求分別有它們自己的子請求及資料。ngx_connection_t中的data欄位保存的是當前可以往out chain發送資料的請求，文章開頭說到發到用戶端的資料必須按照子請求創建的順序發送，這裏即是按後續遍曆的方法（SUB11->DATA11->SUB12->DATA12->(SUB1)->DATA1->SUB21->SUB22->(SUB2)->(ROOT)），上圖中當前能夠往用戶端（out chain）發送資料的請求顯然就是SUB11，如果SUB12提前執行完成，並產生資料DATA121，只要前面它還有節點未發送完畢，DATA121只能先掛載在SUB12的postponed鏈表下。這裏還要注意一下的是c->data的設置，當SUB11執行完並且發送完資料之後，下一個將要發送的節點應該是DATA11，但是該節點實際上保存的是資料，而不是子請求，所以c->data這時應該指向的是擁有改資料節點的SUB1請求。

下面看下源碼具體是怎樣實現的，首先是ngx_http_postpone_filter函數：

.. code-block:: none

    static ngx_int_t
    ngx_http_postpone_filter(ngx_http_request_t *r, ngx_chain_t *in)
    {
        ...
        /* 當前請求不能往out chain發送資料，如果產生了資料，新建一個節點，
           將它保存在當前請求的postponed隊尾。這樣就保證了資料按序發到用戶端 */
        if (r != c->data) {   

            if (in) {
                ngx_http_postpone_filter_add(r, in);
                return NGX_OK;
            }
            ...
            return NGX_OK;
        }
        /* 到這裏，表示當前請求可以往out chain發送資料，如果它的postponed鏈表中沒有子請求，也沒有資料，
           則直接發送當前產生的資料in或者繼續發送out chain中之前沒有發送完成的資料 */
        if (r->postponed == NULL) {  
                                    
            if (in || c->buffered) {
                return ngx_http_next_filter(r->main, in);
            }
            /* 當前請求沒有需要發送的資料 */
            return NGX_OK;
        }
        /* 當前請求的postponed鏈表中之前就存在需要處理的節點，則新建一個節點，保存當前產生的資料in，
           並將它插入到postponed隊尾 */
        if (in) {  
            ngx_http_postpone_filter_add(r, in);
        }
        /* 處理postponed鏈表中的節點 */
        do {   
            pr = r->postponed;
            /* 如果該節點保存的是一個子請求，則將它加到主請求的posted_requests鏈表中，
               以便下次調用ngx_http_run_posted_requests函數，處理該子節點 */
            if (pr->request) {

                ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                               "http postpone filter wake \"%V?%V\"",
                               &pr->request->uri, &pr->request->args);

                r->postponed = pr->next;

                /* 按照後續遍曆產生的序列，因為當前請求（節點）有未處理的子請求(節點)，
                   必須先處理完改子請求，才能繼續處理後面的子節點。
                   這裏將該子請求設置為可以往out chain發送資料的請求。  */
                c->data = pr->request;
                /* 將該子請求加入主請求的posted_requests鏈表 */
                return ngx_http_post_request(pr->request, NULL);
            }
            /* 如果該節點保存的是資料，可以直接處理該節點，將它發送到out chain */
            if (pr->out == NULL) {
                ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                              "http postpone filter NULL output",
                              &r->uri, &r->args);

            } else {
                ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                               "http postpone filter output \"%V?%V\"",
                               &r->uri, &r->args);

                if (ngx_http_next_filter(r->main, pr->out) == NGX_ERROR) {
                    return NGX_ERROR;
                }
            }

            r->postponed = pr->next;

        } while (r->postponed);

        return NGX_OK;
    }

再來看ngx_http_finalzie_request函數：

.. code-block:: none

    void
    ngx_http_finalize_request(ngx_http_request_t *r, ngx_int_t rc) 
    {
      ...
        /* 如果當前請求是一個子請求，檢查它是否有回調handler，有的話執行之 */
        if (r != r->main && r->post_subrequest) {
            rc = r->post_subrequest->handler(r, r->post_subrequest->data, rc);
        }

      ...
        
        /* 子請求 */
        if (r != r->main) {  
            /* 該子請求還有未處理完的資料或者子請求 */
            if (r->buffered || r->postponed) {
                /* 添加一個該子請求的寫事件，並設置合適的write event hander，
                   以便下次寫事件來的時候繼續處理，這裏實際上下次執行時會調用ngx_http_output_filter函數，
                   最終還是會進入ngx_http_postpone_filter進行處理 */
                if (ngx_http_set_write_handler(r) != NGX_OK) {
                    ngx_http_terminate_request(r, 0);
                }

                return;
            }
            ...
                  
            pr = r->parent;
            

            /* 該子請求已經處理完畢，如果它擁有發送資料的權利，則將權利移交給父請求， */
            if (r == c->data) { 

                r->main->count--;

                if (!r->logged) {

                    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

                    if (clcf->log_subrequest) {
                        ngx_http_log_request(r);
                    }

                    r->logged = 1;

                } else {
                    ngx_log_error(NGX_LOG_ALERT, c->log, 0,
                                  "subrequest: \"%V?%V\" logged again",
                                  &r->uri, &r->args);
                }

                r->done = 1;
                /* 如果該子請求不是提前完成，則從父請求的postponed鏈表中刪除 */
                if (pr->postponed && pr->postponed->request == r) {
                    pr->postponed = pr->postponed->next;
                }
                /* 將發送權利移交給父請求，父請求下次執行的時候會發送它的postponed鏈表中可以
                   發送的資料節點，或者將發送權利移交給它的下一個子請求 */
                c->data = pr;   

            } else {
                /* 到這裏其實表明該子請求提前執行完成，而且它沒有產生任何資料，則它下次再次獲得
                   執行機會時，將會執行ngx_http_request_finalzier函數，它實際上是執行
                   ngx_http_finalzie_request（r,0），也就是什麼都不幹，直到輪到它發送資料時，
                   ngx_http_finalzie_request函數會將它從父請求的postponed鏈表中刪除 */
                r->write_event_handler = ngx_http_request_finalizer;

                if (r->waited) {
                    r->done = 1;
                }
            }
            /* 將父請求加入posted_request隊尾，獲得一次運行機會 */
            if (ngx_http_post_request(pr, NULL) != NGX_OK) {
                r->main->count++;
                ngx_http_terminate_request(r, 0);
                return;
            }

            return;
        }
        /* 這裏是處理主請求結束的邏輯，如果主請求有未發送的資料或者未處理的子請求，
           則給主請求添加寫事件，並設置合適的write event hander，
           以便下次寫事件來的時候繼續處理 */
        if (r->buffered || c->buffered || r->postponed || r->blocked) {

            if (ngx_http_set_write_handler(r) != NGX_OK) {
                ngx_http_terminate_request(r, 0);
            }

            return;
        }

     ...
    } 

