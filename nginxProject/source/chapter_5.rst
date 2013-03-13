upstream模組
======================

upstream模組 (100%)
-----------------------

nginx模組一般被分成三大類：handler、filter和upstream。前面的章節中，讀者已經瞭解了handler、filter。利用這兩類模組，可以使nginx輕鬆完成任何單機工作。而本章介紹的upstream，將使nginx將跨越單機的限制，完成網路資料的接收、處理和轉發。

資料轉發功能，為nginx提供了跨越單機的橫向處理能力，使nginx擺脫只能為終端節點提供單一功能的限制，而使它具備了網路應用級別的拆分、封裝和整合的戰略功能。在雲模型大行其道的今天，資料轉發使nginx有能力構建一個網路應用的關鍵元件。當然，一個網路應用的關鍵元件往往一開始都會考慮通過高級開發語言編寫，因為開發比較方便，但系統到達一定規模，需要更重視性能的時候，這些高階語言為了達成目標所做的結構化修改所付出的代價會使nginx的upstream模組就呈現出極大的吸引力，因為他天生就快。作為附帶，nginx的配置提供的層次化和松耦合使得系統的擴展性也可能達到比較高的程度。

言歸正傳，下面介紹upstream的寫法。

upstream模組介面
+++++++++++++++++++++++++++

從本質上說，upstream屬於handler，只是他不產生自己的內容，而是通過請求後端伺服器得到內容，所以才稱為upstream（上游）。請求並取得回應內容的整個過程已經被封裝到nginx內部，所以upstream模組只需要開發若干回調函數，完成構造請求和解析回應等具體的工作。

這些回調函數如下表所示：

+-------------------+--------------------------------------------------------------+
|create_request     |生成發送到後端伺服器的請求緩衝（緩衝鏈）。                    |
+-------------------+--------------------------------------------------------------+
|reinit_request     |在某台後端伺服器出錯的情況，nginx會嘗試另一台後端伺服器。     |
|                   |nginx選定新的伺服器以後，會先調用此函數，然後再次調用         |
|                   |create_request，以重新初始化upstream模組的工作狀態。          |
+-------------------+--------------------------------------------------------------+
|process_header     |處理後端伺服器返回的資訊頭部。所謂頭部是與upstream server     |
|                   |通信的協定規定的，比如HTTP協定的header部分，或者memcached     |
|                   |協定的回應狀態部分。                                          |
+-------------------+--------------------------------------------------------------+
|abort_request      |在用戶端放棄請求時被調用。不需要在函數中實現關閉後端服務      |
|                   |器連接的功能，系統會自動完成關閉連接的步驟，所以一般此函      |
|                   |數不會進行任何具體工作。                                      |
+-------------------+--------------------------------------------------------------+
|finalize_request   |正常完成與後端伺服器的請求後調用該函數，與abort_request       |
|                   |相同，一般也不會進行任何具體工作。                            |
+-------------------+--------------------------------------------------------------+
|input_filter       |處理後端伺服器返回的回應正文。nginx默認的input_filter會       |
|                   |將收到的內容封裝成為緩衝區鏈ngx_chain。該鏈由upstream的       |
|                   |out_bufs指標域定位，所以開發人員可以在模組以外通過該指標      |
|                   |得到後端伺服器返回的正文資料。memcached模組實現了自己的       |
|                   |input_filter，在後面會具體分析這個模組。                      |
+-------------------+--------------------------------------------------------------+
|input_filter_init  |初始化input filter的上下文。nginx默認的input_filter_init      |
|                   |直接返回。                                                    |
+-------------------+--------------------------------------------------------------+

memcached模組分析
++++++++++++++++++++++++++++++

memcache是一款高性能的分散式cache系統，得到了非常廣泛的應用。memcache定義了一套私有通信協議，使得不能通過HTTP請求來訪問memcache。但協議本身簡單高效，而且memcache使用廣泛，所以大部分現代開發語言和平臺都提供了memcache支援，方便開發者使用memcache。

nginx提供了ngx_http_memcached模組，提供從memcache讀取資料的功能，而不提供向memcache寫資料的功能。作為web伺服器，這種設計是可以接受的。

下面，我們開始分析ngx_http_memcached模組，一窺upstream的奧秘。

Handler模組？
^^^^^^^^^^^^^^^^^^^^^^^^

初看memcached模組，大家可能覺得並無特別之處。如果稍微細看，甚至覺得有點像handler模組，當大家看到這段代碼以後，必定疑惑為什麼會跟handler模組一模一樣。

.. code-block:: none

        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        clcf->handler = ngx_http_memcached_handler;

因為upstream模組使用的就是handler模組的接入方式。同時，upstream模組的指令系統的設計也是遵循handler模組的基本規則：配置該模組才會執行該模組。

.. code-block:: none

        { ngx_string("memcached_pass"),
          NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
          ngx_http_memcached_pass,
          NGX_HTTP_LOC_CONF_OFFSET,
          0,
          NULL }

所以大家覺得眼熟是好事，說明大家對Handler的寫法已經很熟悉了。

Upstream模組！
^^^^^^^^^^^^^^^^^^^^^^^^^^

那麼，upstream模組的特別之處究竟在哪里呢？答案是就在模組處理函數的實現中。upstream模組的處理函數進行的操作都包含一個固定的流程。在memcached的例子中，可以觀察ngx_http_memcached_handler的代碼，可以發現，這個固定的操作流程是：

1\. 創建upstream資料結構。

.. code-block:: none

        if (ngx_http_upstream_create(r) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

2\. 設置模組的tag和schema。schema現在只會用於日誌，tag會用於buf_chain管理。

.. code-block:: none

        u = r->upstream;

        ngx_str_set(&u->schema, "memcached://");
        u->output.tag = (ngx_buf_tag_t) &ngx_http_memcached_module;

3\. 設置upstream的後端伺服器列表資料結構。

.. code-block:: none

        mlcf = ngx_http_get_module_loc_conf(r, ngx_http_memcached_module);
        u->conf = &mlcf->upstream;

4\. 設置upstream回調函數。在這裏列出的代碼稍稍調整了代碼順序。

.. code-block:: none

        u->create_request = ngx_http_memcached_create_request;
        u->reinit_request = ngx_http_memcached_reinit_request;
        u->process_header = ngx_http_memcached_process_header;
        u->abort_request = ngx_http_memcached_abort_request;
        u->finalize_request = ngx_http_memcached_finalize_request;
        u->input_filter_init = ngx_http_memcached_filter_init;
        u->input_filter = ngx_http_memcached_filter;

5\. 創建並設置upstream環境資料結構。

.. code-block:: none 

        ctx = ngx_palloc(r->pool, sizeof(ngx_http_memcached_ctx_t));
        if (ctx == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        ctx->rest = NGX_HTTP_MEMCACHED_END;
        ctx->request = r;

        ngx_http_set_ctx(r, ctx, ngx_http_memcached_module);

        u->input_filter_ctx = ctx;

6\. 完成upstream初始化並進行收尾工作。

.. code-block:: none

        r->main->count++;
        ngx_http_upstream_init(r);
        return NGX_DONE;

任何upstream模組，簡單如memcached，複雜如proxy、fastcgi都是如此。不同的upstream模組在這6步中的最大差別會出現在第2、3、4、5上。其中第2、4兩步很容易理解，不同的模組設置的標誌和使用的回調函數肯定不同。第5步也不難理解，只有第3步是最為晦澀的，不同的模組在取得後端伺服器列表時，策略的差異非常大，有如memcached這樣簡單明瞭的，也有如proxy那樣邏輯複雜的。這個問題先記下來，等把memcached剖析清楚了，再單獨討論。

第6步是一個常態。將count加1，然後返回NGX_DONE。nginx遇到這種情況，雖然會認為當前請求的處理已經結束，但是不會釋放請求使用的記憶體資源，也不會關閉與用戶端的連接。之所以需要這樣，是因為nginx建立了upstream請求和用戶端請求之間一對一的關係，在後續使用ngx_event_pipe將upstream回應發送回用戶端時，還要使用到這些保存著用戶端資訊的資料結構。這部分會在後面的原理篇做具體介紹，這裏不再展開。

將upstream請求和用戶端請求進行一對一綁定，這個設計有優勢也有缺陷。優勢就是簡化模組開發，可以將精力集中在模組邏輯上，而缺陷同樣明顯，一對一的設計很多時候都不能滿足複雜邏輯的需要。對於這一點，將會在後面的原理篇來闡述。


回調函數
^^^^^^^^^^^^^^^^^^^^^

前面剖析了memcached模組的骨架，現在開始逐個解決每個回調函數。

1\. ngx_http_memcached_create_request：很簡單的按照設置的內容生成一個key，接著生成一個“get $key”的請求，放在r->upstream->request_bufs裏面。

2\. ngx_http_memcached_reinit_request：無需初始化。

3\. ngx_http_memcached_abort_request：無需額外操作。

4\. ngx_http_memcached_finalize_request：無需額外操作。

5\. ngx_http_memcached_process_header：模組的業務重點函數。memcache協定將頭部資訊被定義為第一行文本，可以找到這段代碼證明：

.. code-block:: none

        for (p = u->buffer.pos; p < u->buffer.last; p++) {
            if ( * p == LF) {
            goto found;
        }

如果在已讀入緩衝的資料中沒有發現LF('\n')字元，函數返回NGX_AGAIN，表示頭部未完全讀入，需要繼續讀取資料。nginx在收到新的資料以後會再次調用該函數。

nginx處理後端伺服器的回應頭時只會使用一塊緩存，所有資料都在這塊緩存中，所以解析頭部資訊時不需要考慮頭部資訊跨越多塊緩存的情況。而如果頭部過大，不能保存在這塊緩存中，nginx會返回錯誤資訊給用戶端，並記錄error log，提示緩存不夠大。

process_header的重要職責是將後端伺服器返回的狀態翻譯成返回給用戶端的狀態。例如，在ngx_http_memcached_process_header中，有這樣幾段代碼：

.. code-block:: none

        r->headers_out.content_length_n = ngx_atoof(len, p - len - 1);

        u->headers_in.status_n = 200;
        u->state->status = 200;

        u->headers_in.status_n = 404;
        u->state->status = 404;

u->state用於計算upstream相關的變數。比如u->status->status將被用於計算變數“upstream_status”的值。u->headers_in將被作為返回給用戶端的回應返回狀態碼。而第一行則是設置返回給用戶端的回應的長度。

在這個函數中不能忘記的一件事情是處理完頭部資訊以後需要將讀指標pos後移，否則這段資料也將被複製到返回給用戶端的回應的正文中，進而導致正文內容不正確。

.. code-block:: none

        u->buffer.pos = p + 1;

process_header函數完成回應頭的正確處理，應該返回NGX_OK。如果返回NGX_AGAIN，表示未讀取完整資料，需要從後端伺服器繼續讀取資料。返回NGX_DECLINED無意義，其他任何返回值都被認為是出錯狀態，nginx將結束upstream請求並返回錯誤資訊。

6\. ngx_http_memcached_filter_init：修正從後端伺服器收到的內容長度。因為在處理header時沒有加上這部分長度。

7\. ngx_http_memcached_filter：memcached模組是少有的帶有處理正文的回調函數的模組。因為memcached模組需要過濾正文末尾CRLF "END" CRLF，所以實現了自己的filter回調函數。處理正文的實際意義是將從後端伺服器收到的正文有效內容封裝成ngx_chain_t，並加在u->out_bufs末尾。nginx並不進行資料拷貝，而是建立ngx_buf_t資料結構指向這些資料記憶體區，然後由ngx_chain_t組織這些buf。這種實現避免了記憶體大量搬遷，也是nginx高效的奧秘之一。

本節回顧
+++++++++++++++++++++

這一節介紹了upstream模組的基本組成。upstream模組是從handler模組發展而來，指令系統和模組生效方式與handler模組無異。不同之處在於，upstream模組在handler函數中設置眾多回調函數。實際工作都是由這些回調函數完成的。每個回調函數都是在upstream的某個固定階段執行，各司其職，大部分回調函數一般不會真正用到。upstream最重要的回調函數是create_request、process_header和input_filter，他們共同實現了與後端伺服器的協定的解析部分。


負載均衡模組 (100%)
-----------------------

負載均衡模組用於從"upstream"指令定義的後端主機列表中選取一台主機。nginx先使用負載均衡模組找到一台主機，再使用upstream模組實現與這台主機的交互。為了方便介紹負載均衡模組，做到言之有物，以下選取nginx內置的ip hash模組作為實際例子進行分析。

配置
++++++++++++++

要瞭解負載均衡模組的開發方法，首先需要瞭解負載均衡模組的使用方法。因為負載均衡模組與之前書中提到的模組差別比較大，所以我們從配置入手比較容易理解。

在配置檔中，我們如果需要使用ip hash的負載均衡演算法。我們需要寫一個類似下面的配置：

.. code-block:: none

        upstream test {
            ip_hash;

            server 192.168.0.1;
            server 192.168.0.2;
        }

從配置我們可以看出負載均衡模組的使用場景：
1\. 核心指令"ip_hash"只能在upstream {}中使用。這條指令用於通知nginx使用ip hash負載均衡演算法。如果沒加這條指令，nginx會使用默認的round robin負載均衡模組。請各位讀者對比handler模組的配置，是不是有共同點？
2\. upstream {}中的指令可能出現在"server"指令前，可能出現在"server"指令後，也可能出現在兩條"server"指令之間。各位讀者可能會有疑問，有什麼差別麼？那麼請各位讀者嘗試下面這個配置：

.. code-block:: none

        upstream test {
            server 192.168.0.1 weight=5;
            ip_hash;
            server 192.168.0.2 weight=7;
        }

神奇的事情出現了：

.. code-block:: none

        nginx: [emerg] invalid parameter "weight=7" in nginx.conf:103
        configuration file nginx.conf test failed

可見ip_hash指令的確能影響到配置的解析。

指令
+++++++++++++++++

配置決定指令系統，現在就來看ip_hash的指令定義：

.. code-block:: none

    static ngx_command_t  ngx_http_upstream_ip_hash_commands[] = {

        { ngx_string("ip_hash"),
          NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
          ngx_http_upstream_ip_hash,
          0,
          0,
          NULL },

        ngx_null_command
    };

沒有特別的東西，除了指令屬性是NGX_HTTP_UPS_CONF。這個屬性表示該指令的適用範圍是upstream{}。

鉤子
+++++++++++++++++

以從前面的章節得到的經驗，大家應該知道這裏就是模組的切入點了。負載均衡模組的鉤子代碼都是有規律的，這裏通過ip_hash模組來分析這個規律。

.. code-block:: none

    static char *
    ngx_http_upstream_ip_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
    {
        ngx_http_upstream_srv_conf_t  *uscf;

        uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

        uscf->peer.init_upstream = ngx_http_upstream_init_ip_hash;

        uscf->flags = NGX_HTTP_UPSTREAM_CREATE
                    |NGX_HTTP_UPSTREAM_MAX_FAILS
                    |NGX_HTTP_UPSTREAM_FAIL_TIMEOUT
                    |NGX_HTTP_UPSTREAM_DOWN;

        return NGX_CONF_OK;
    }

這段代碼中有兩點值得我們注意。一個是uscf->flags的設置，另一個是設置init_upstream回調。

設置uscf->flags
^^^^^^^^^^^^^^^^^^^^^^^^^^

1. NGX_HTTP_UPSTREAM_CREATE：創建標誌，如果含有創建標誌的話，nginx會檢查重複創建，以及必要參數是否填寫；

2. NGX_HTTP_UPSTREAM_MAX_FAILS：可以在server中使用max_fails屬性；

3. NGX_HTTP_UPSTREAM_FAIL_TIMEOUT：可以在server中使用fail_timeout屬性；

4. NGX_HTTP_UPSTREAM_DOWN：可以在server中使用down屬性；

此外還有下面屬性：

5. NGX_HTTP_UPSTREAM_WEIGHT：可以在server中使用weight屬性；

6. NGX_HTTP_UPSTREAM_BACKUP：可以在server中使用backup屬性。

聰明的讀者如果聯想到剛剛遇到的那個神奇的配置錯誤，可以得出一個結論：在負載均衡模組的指令處理函數中可以設置並修改upstream{}中"server"指令支援的屬性。這是一個很重要的性質，因為不同的負載均衡模組對各種屬性的支援情況都是不一樣的，那麼就需要在解析配置檔的時候檢測出是否使用了不支援的負載均衡屬性並給出錯誤提示，這對於提升系統維護性是很有意義的。但是，這種機制也存在缺陷，正如前面的例子所示，沒有機制能夠追加檢查在更新支援屬性之前已經配置了不支援屬性的"server"指令。

設置init_upstream回調
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

nginx初始化upstream時，會在ngx_http_upstream_init_main_conf函數中調用設置的回調函數初始化負載均衡模組。這裏不太好理解的是uscf的具體位置。通過下面的示意圖，說明upstream負載均衡模組的配置的記憶體佈局。

.. image:: /images/chapter-5-1.PNG

從圖上可以看出，MAIN_CONF中ngx_upstream_module模組的配置項中有一個指標陣列upstreams，陣列中的每個元素對應就是配置檔中每一個upstream{}的資訊。更具體的將會在後面的原理篇討論。

初始化配置
++++++++++++++++++++++++

init_upstream回調函數執行時需要初始化負載均衡模組的配置，還要設置一個新鉤子，這個鉤子函數會在nginx處理每個請求時作為初始化函數調用，關於這個新鉤子函數的功能，後面會有詳細的描述。這裏，我們先分析IP hash模組初始化配置的代碼：

.. code-block:: none

    ngx_http_upstream_init_round_robin(cf, us);
    us->peer.init = ngx_http_upstream_init_ip_hash_peer;

這段代碼非常簡單：IP hash模組首先調用另一個負載均衡模組Round Robin的初始化函數，然後再設置自己的處理請求階段初始化鉤子。實際上幾個負載均衡模組可以組成一條鏈表，每次都是從鏈首的模組開始進行處理。如果模組決定不處理，可以將處理權交給鏈表中的下一個模組。這裏，IP hash模組指定Round Robin模組作為自己的後繼負載均衡模組，所以在自己的初始化配置函數中也對Round Robin模組進行初始化。

初始化請求
++++++++++++++++++++++++

nginx收到一個請求以後，如果發現需要訪問upstream，就會執行對應的peer.init函數。這是在初始化配置時設置的回調函數。這個函數最重要的作用是構造一張表，當前請求可以使用的upstream伺服器被依次添加到這張表中。之所以需要這張表，最重要的原因是如果upstream伺服器出現異常，不能提供服務時，可以從這張表中取得其他伺服器進行重試操作。此外，這張表也可以用於負載均衡的計算。之所以構造這張表的行為放在這裏而不是在前面初始化配置的階段，是因為upstream需要為每一個請求提供獨立隔離的環境。

為了討論peer.init的核心，我們還是看IP hash模組的實現：

.. code-block:: none

    r->upstream->peer.data = &iphp->rrp;

    ngx_http_upstream_init_round_robin_peer(r, us);

    r->upstream->peer.get = ngx_http_upstream_get_ip_hash_peer;

第一行是設置資料指標，這個指標就是指向前面提到的那張表；

第二行是調用Round Robin模組的回調函數對該模組進行請求初始化。面前已經提到，一個負載均衡模組可以調用其他負載均衡模組以提供功能的補充。

第三行是設置一個新的回調函數get。該函數負責從表中取出某個伺服器。除了get回調函數，還有另一個r->upstream->peer.free的回調函數。該函數在upstream請求完成後調用，負責做一些善後工作。比如我們需要維護一個upstream伺服器訪問計數器，那麼可以在get函數中對其加1，在free中對其減1。如果是SSL的話，nginx還提供兩個回調函數peer.set_session和peer.save_session。一般來說，有兩個切入點實現負載均衡演算法，其一是在這裏，其二是在get回調函數中。

peer.get和peer.free回調函數
+++++++++++++++++++++++++++++++++

這兩個函數是負載均衡模組最底層的函數，負責實際獲取一個連接和回收一個連接的預備操作。之所以說是預備操作，是因為在這兩個函數中，並不實際進行建立連接或者釋放連接的動作，而只是執行獲取連接的位址或維護連接狀態的操作。需要理解的清楚一點，在peer.get函數中獲取連接的位址資訊，並不代表這時連接一定沒有被建立，相反的，通過get函數的返回值，nginx可以瞭解是否存在可用連接，連接是否已經建立。這些返回值總結如下：

+-------------------+-------------------------------------------+-----------------------------------------+
|返回值             |說明                                       |nginx後續動作                            |
+-------------------+-------------------------------------------+-----------------------------------------+
|NGX_DONE           |得到了連接位址資訊，並且連接已經建立。     |直接使用連接，發送資料。                 |
+-------------------+-------------------------------------------+-----------------------------------------+
|NGX_OK             |得到了連接位址資訊，但連接並未建立。       |建立連接，如連接不能立即建立，設置事件， |
|                   |                                           |暫停執行本請求，執行別的請求。           |
+-------------------+-------------------------------------------+-----------------------------------------+
|NGX_BUSY           |所有連接均不可用。                         |返回502錯誤至用戶端。                    |
+-------------------+-------------------------------------------+-----------------------------------------+

各位讀者看到上面這張表，可能會有幾個問題浮現出來：

:Q: 什麼時候連接是已經建立的？
:A: 使用後端keepalive連接的時候，連接在使用完以後並不關閉，而是存放在一個佇列中，新的請求只需要從佇列中取出連接，這些連接都是已經準備好的。

:Q: 什麼叫所有連接均不可用？
:A: 初始化請求的過程中，建立了一張表，get函數負責每次從這張表中不重複的取出一個連接，當無法從表中取得一個新的連接時，即所有連接均不可用。

:Q: 對於一個請求，peer.get函數可能被調用多次麼？
:A: 正式如此。當某次peer.get函數得到的連接位址連接不上，或者請求對應的伺服器得到異常回應，nginx會執行ngx_http_upstream_next，然後可能再次調用peer.get函數嘗試別的連接。upstream整體流程如下：

.. image:: /images/chapter-5-2.PNG

本節回顧
+++++++++++++++++++++

這一節介紹了負載均衡模組的基本組成。負載均衡模組的配置區集中在upstream{}塊中。負載均衡模組的回調函數體系是以init_upstream為起點，經歷init_peer，最終到達peer.get和peer.free。其中init_peer負責建立每個請求使用的server列表，peer.get負責從server列表中選擇某個server（一般是不重複選擇），而peer.free負責server釋放前的資源釋放工作。最後，這一節通過一張圖將upstream模組和負載均衡模組在請求處理過程中的相互關係展現出來。
