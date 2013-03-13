模組開發
============

upstream模組
------------

nginx模組一般被分成三大類：handler、filter和upstream。前面的章節中，讀者已經瞭解了handler、filter。利用這兩類模組，可以使nginx輕鬆完成任何單機工作。而本章介紹的upstream，將使nginx將跨越單機的限制，完成網路資料的接收、處理和轉發。

資料轉發功能，為nginx提供了跨越單機的橫向處理能力，使nginx擺脫只能為終端節點提供單一功能的限制，而使它具備了網路應用級別的拆分、封裝和整合的戰略功能。在雲模型大行其道的今天，資料轉發使nginx有能力構建一個網路應用的關鍵元件。當然，一個網路應用的關鍵元件往往一開始都會考慮通過高級開發語言編寫，因為開發比較方便，但系統到達一定規模，需要更重視性能的時候，這些高階語言為了達成目標所做的結構化修改所付出的代價會使nginx的upstream模組就呈現出極大的吸引力，因為他天生就快。作為附帶，nginx的配置提供的層次化和松耦合使得系統的擴展性也可能達到比較高的程度。

言歸正傳，下面介紹upstream的寫法。

upstream模組介面
+++++++++++++++++++

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
^^^^^^^^^^^^^^^

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
^^^^^^^^^^^^^^^

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
^^^^^^^^^^^

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

本節小結
++++++++++++

在這一節裏，大家對upstream模組的基本組成有了一些認識。upstream模組是從handler模組發展而來，指令系統和模組生效方式與handler模組無異。不同之處在於，upstream模組在handler函數中設置眾多回調函數。實際工作都是由這些回調函數完成的。每個回調函數都是在upstream的某個固定階段執行，各司其職，大部分回調函數一般不會真正用到。upstream最重要的回調函數是create_request、process_header和input_filter，他們共同實現了與後端伺服器的協定的解析部分。
