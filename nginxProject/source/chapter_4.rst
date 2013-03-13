過濾模組 (90%)
======================

過濾模組簡介 (90%)
------------------------

執行時間和內容 (90%)
+++++++++++++++++++++++++++
過濾（filter）模組是過濾回應頭和內容的模組，可以對回復的頭和內容進行處理。它的處理時間在獲取回復內容之後，向用戶發送請求之前。它的處理過程分為兩個階段，過濾HTTP回復的頭部和主體，在這兩個階段可以分別對頭部和主體進行修改。

在代碼中有類似的函數：

.. code-block:: none

		ngx_http_top_header_filter(r);
		ngx_http_top_body_filter(r, in);

就是分別對頭部和主體進行過濾的函數。所有模組的回應內容要返回給用戶端，都必須調用這兩個介面。


執行順序 (90%)
+++++++++++++++++++++

過濾模組的調用是有順序的，它的順序在編譯的時候就決定了。控制編譯的腳本位於auto/modules中，當你編譯完Nginx以後，可以在objs目錄下面看到一個ngx_modules.c的文件。打開這個檔，有類似的代碼：

.. code-block:: none

		ngx_module_t *ngx_modules[] = {
			...
			&ngx_http_write_filter_module,
			&ngx_http_header_filter_module,
			&ngx_http_chunked_filter_module,
			&ngx_http_range_header_filter_module,
			&ngx_http_gzip_filter_module,
			&ngx_http_postpone_filter_module,
			&ngx_http_ssi_filter_module,
			&ngx_http_charset_filter_module,
			&ngx_http_userid_filter_module,
			&ngx_http_headers_filter_module,
			&ngx_http_copy_filter_module,
			&ngx_http_range_body_filter_module,
			&ngx_http_not_modified_filter_module,
			NULL
		};

從write_filter到not_modified_filter，模組的執行順序是反向的。也就是說最早執行的是not_modified_filter，然後各個模組依次執行。所有第三方的模組只能加入到copy_filter和headers_filter模組之間執行。

Nginx執行的時候是怎麼按照次序依次來執行各個過濾模組呢？它採用了一種很隱晦的方法，即通過局部的總體變數。比如，在每個filter模組，很可能看到如下代碼：

.. code-block:: none

		static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
		static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;
		
		...

		ngx_http_next_header_filter = ngx_http_top_header_filter;
		ngx_http_top_header_filter = ngx_http_example_header_filter;

		ngx_http_next_body_filter = ngx_http_top_body_filter;
		ngx_http_top_body_filter = ngx_http_example_body_filter;

ngx_http_top_header_filter是一個總體變數。當編譯進一個filter模組的時候，就被賦值為當前filter模組的處理函數。而ngx_http_next_header_filter是一個局部總體變數，它保存了編譯前上一個filter模組的處理函數。所以整體看來，就像用總體變數組成的一條單向鏈表。

每個模組想執行下一個過濾函數，只要調用一下ngx_http_next_header_filter這個局部變數。而整個過濾模組鏈的入口，需要調用ngx_http_top_header_filter這個總體變數。ngx_http_top_body_filter的行為與header fitler類似。

回應頭和回應體過濾函數的執行順序如下所示：

.. image:: /images/chapter-4-1.png

這圖只表示了head_filter和body_filter之間的執行順序，在header_filter和body_filter處理函數之間，在body_filter處理函數之間，可能還有其他執行代碼。

模組編譯 (90%)
++++++++++++++++++++

Nginx可以方便的加入第三方的過濾模組。在過濾模組的目錄裏，首先需要加入config檔，檔的內容如下：

.. code-block:: none

		ngx_addon_name=ngx_http_example_filter_module
		HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_example_filter_module"
		NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_example_filter_module.c"

說明把這個名為ngx_http_example_filter_module的過濾模組加入，ngx_http_example_filter_module.c是該模組的源代碼。

注意HTTP_AUX_FILTER_MODULES這個變數與一般的內容處理模組不同。


過濾模組的分析 (90%)
--------------------------

相關結構體 (90%)
+++++++++++++++++++++
ngx_chain_t 結構非常簡單，是一個單向鏈表：

.. code-block:: none
        
        typedef struct ngx_chain_s ngx_chain_t;
         
		struct ngx_chain_s {
			ngx_buf_t    *buf;
			ngx_chain_t  *next;
		};

在過濾模組中，所有輸出的內容都是通過一條單向鏈表所組成。這種單向鏈表的設計，正好應和了Nginx流式的輸出模式。每次Nginx都是讀到一部分的內容，就放到鏈表，然後輸出出去。這種設計的好處是簡單，非阻塞，但是相應的問題就是跨鏈表的內容操作非常麻煩，如果需要跨鏈表，很多時候都只能緩存鏈表的內容。

單鏈表負載的就是ngx_buf_t，這個結構體使用非常廣泛，先讓我們看下該結構體的代碼：

.. code-block:: none 

		struct ngx_buf_s {
			u_char          *pos;       /* 當前buffer真實內容的起始位置 */
			u_char          *last;      /* 當前buffer真實內容的結束位置 */
			off_t            file_pos;  /* 在檔中真實內容的起始位置   */
			off_t            file_last; /* 在檔中真實內容的結束位置   */

			u_char          *start;    /* buffer記憶體的開始分配的位置 */
			u_char          *end;      /* buffer記憶體的結束分配的位置 */
			ngx_buf_tag_t    tag;      /* buffer屬於哪個模組的標誌 */
			ngx_file_t      *file;     /* buffer所引用的文件 */

	 		/* 用來引用替換過後的buffer，以便當所有buffer輸出以後，
			 * 這個影子buffer可以被釋放。
			 */
			ngx_buf_t       *shadow; 

			/* the buf's content could be changed */
			unsigned         temporary:1;

			/*
			 * the buf's content is in a memory cache or in a read only memory
			 * and must not be changed
			 */
			unsigned         memory:1;

			/* the buf's content is mmap()ed and must not be changed */
			unsigned         mmap:1;

			unsigned         recycled:1; /* 記憶體可以被輸出並回收 */
			unsigned         in_file:1;  /* buffer的內容在檔中 */
			/* 馬上全部輸出buffer的內容, gzip模組裏面用得比較多 */
			unsigned         flush:1;
			/* 基本上是一段輸出鏈的最後一個buffer帶的標誌，標示可以輸出，
			 * 有些零長度的buffer也可以置該標誌
			 */
			unsigned         sync:1;
			/* 所有請求裏面最後一塊buffer，包含子請求 */
			unsigned         last_buf:1;
			/* 當前請求輸出鏈的最後一塊buffer         */
			unsigned         last_in_chain:1;
			/* shadow鏈裏面的最後buffer，可以釋放buffer了 */
			unsigned         last_shadow:1;
			/* 是否是暫存檔 */
			unsigned         temp_file:1;

			/* 統計用，表示使用次數 */
			/* STUB */ int   num;
		};

一般buffer結構體可以表示一塊記憶體，記憶體的起始和結束位址分別用start和end表示，pos和last表示實際的內容。如果內容已經處理過了，pos的位置就可以往後移動。如果讀取到新的內容，last的位置就會往後移動。所以buffer可以在多次調用過程中使用。如果last等於end，就說明這塊記憶體已經用完了。如果pos等於last，說明記憶體已經處理完了。下面是一個簡單的示意圖，說明buffer中指標的用法：

.. image:: /images/chapter-4-2.png


回應頭過濾函數 (90%)
+++++++++++++++++++++++++

回應頭過濾函數主要的用處就是處理HTTP回應的頭，可以根據實際情況對於回應頭進行修改或者添加刪除。回應頭過濾函數先於回應體過濾函數，而且只調用一次，所以一般可作過濾模組的初始化工作。

響應頭過濾函數的入口只有一個：

.. code-block:: none

		ngx_int_t
		ngx_http_send_header(ngx_http_request_t *r)
		{
			...

			return ngx_http_top_header_filter(r);
		}

該函數向用戶端發送回復的時候調用，然後按前一節所述的執行順序。該函數的返回值一般是NGX_OK，NGX_ERROR和NGX_AGAIN，分別表示處理成功，失敗和未完成。

你可以把HTTP響應頭的存儲方式想像成一個hash表，在Nginx內部可以很方便地查找和修改各個回應頭部，ngx_http_header_filter_module過濾模組把所有的HTTP頭組合成一個完整的buffer，最終ngx_http_write_filter_module過濾模組把buffer輸出。

按照前一節過濾模組的順序，依次講解如下：

=====================================  ================================================================================================================= 
filter module                           description
=====================================  =================================================================================================================
ngx_http_not_modified_filter_module    默認打開，如果請求的if-modified-since等於回復的last-modified間值，說明回復沒有變化，清空所有回復的內容，返回304。
ngx_http_range_body_filter_module      默認打開，只是回應體過濾函數，支援range功能，如果請求包含range請求，那就只發送range請求的一段內容。
ngx_http_copy_filter_module            始終打開，只是回應體過濾函數， 主要工作是把檔中內容讀到記憶體中，以便進行處理。
ngx_http_headers_filter_module         始終打開，可以設置expire和Cache-control頭，可以添加任意名稱的頭
ngx_http_userid_filter_module          默認關閉，可以添加統計用的識別用戶的cookie。
ngx_http_charset_filter_module         默認關閉，可以添加charset，也可以將內容從一種字元集轉換到另外一種字元集，不支持多位元組字元集。
ngx_http_ssi_filter_module             默認關閉，過濾SSI請求，可以發起子請求，去獲取include進來的檔
ngx_http_postpone_filter_module        始終打開，用來將子請求和主請求的輸出鏈合併
ngx_http_gzip_filter_module            默認關閉，支援流式的壓縮內容
ngx_http_range_header_filter_module    默認打開，只是響應頭過濾函數，用來解析range頭，並產生range響應的頭。
ngx_http_chunked_filter_module         默認打開，對於HTTP/1.1和缺少content-length的回復自動打開。
ngx_http_header_filter_module          始終打開，用來將所有header組成一個完整的HTTP頭。
ngx_http_write_filter_module           始終打開，將輸出鏈拷貝到r->out中，然後輸出內容。
=====================================  ================================================================================================================= 


回應體過濾函數 (90%)
++++++++++++++++++++++++++

回應體過濾函數是過濾回應主體的函數。ngx_http_top_body_filter這個函數每個請求可能會被執行多次，它的入口函數是ngx_http_output_filter，比如：

.. code-block:: none

        ngx_int_t
        ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in)
        {
            ngx_int_t          rc;
            ngx_connection_t  *c;

            c = r->connection;

            rc = ngx_http_top_body_filter(r, in);

            if (rc == NGX_ERROR) {
                /* NGX_ERROR may be returned by any filter */
                c->error = 1;
            }

            return rc;
        }

ngx_http_output_filter可以被一般的靜態處理模組調用，也有可能是在upstream模組裏面被調用，對於整個請求的處理階段來說，他們處於的用處都是一樣的，就是把回應內容過濾，然後發給用戶端。

具體模組的回應體過濾函數的格式類似這樣：

.. code-block:: none

		static int 
		ngx_http_example_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
		{
			...
			
			return ngx_http_next_body_filter(r, in);
		}

該函數的返回值一般是NGX_OK，NGX_ERROR和NGX_AGAIN，分別表示處理成功，失敗和未完成。
        
主要功能介紹 (90%)
^^^^^^^^^^^^^^^^^^^^^^^	
回應的主體內容就存于單鏈表in，鏈表一般不會太長，有時in參數可能為NULL。in中存有buf結構體中，對於靜態檔，這個buf大小默認是32K；對於反向代理的應用，這個buf可能是4k或者8k。為了保持記憶體的低消耗，Nginx一般不會分配過大的記憶體，處理的原則是收到一定的資料，就發送出去。一個簡單的例子，可以看看Nginx的chunked_filter模組，在沒有content-length的情況下，chunk模組可以流式（stream）的加上長度，方便流覽器接收和顯示內容。

在回應體過濾模組中，尤其要注意的是buf的標誌位元，完整描述可以在“相關結構體”這個節中看到。如果buf中包含last標誌，說明是最後一塊buf，可以直接輸出並結束請求了。如果有flush標誌，說明這塊buf需要馬上輸出，不能緩存。如果整塊buffer經過處理完以後，沒有資料了，你可以把buffer的sync標誌置上，表示只是同步的用處。

當所有的過濾模組都處理完畢時，在最後的write_fitler模組中，Nginx會將in輸出鏈拷貝到r->out輸出鏈的末尾，然後調用sendfile或者writev介面輸出。由於Nginx是非阻塞的socket介面，寫操作並不一定會成功，可能會有部分資料還殘存在r->out。在下次的調用中，Nginx會繼續嘗試發送，直至成功。


發出子請求 (90%)
^^^^^^^^^^^^^^^^^^^^^
Nginx過濾模組一大特色就是可以發出子請求，也就是在過濾回應內容的時候，你可以發送新的請求，Nginx會根據你調用的先後順序，將多個回復的內容拼接成正常的響應主體。一個簡單的例子可以參考addtion模組。

Nginx是如何保證父請求和子請求的順序呢？當Nginx發出子請求時，就會調用ngx_http_subrequest函數，將子請求插入父請求的r->postponed鏈表中。子請求會在主請求執行完畢時獲得依次調用。子請求同樣會有一個請求所有的生存期和處理過程，也會進入過濾模組流程。

關鍵點是在postpone_filter模組中，它會拼接主請求和子請求的回應內容。r->postponed按次序保存有父請求和子請求，它是一個鏈表，如果前面一個請求未完成，那後一個請求內容就不會輸出。當前一個請求完成時並輸出時，後一個請求才可輸出，當所有的子請求都完成時，所有的回應內容也就輸出完畢了。


一些優化措施 (90%)
^^^^^^^^^^^^^^^^^^^^^^
Nginx過濾模組涉及到的結構體，主要就是chain和buf，非常簡單。在日常的過濾模組中，這兩類結構使用非常頻繁，Nginx採用類似freelist重複利用的原則，將使用完畢的chain或者buf結構體，放置到一個固定的空閒鏈表裏，以待下次使用。

比如，在通用記憶體池結構體中，pool->chain變數裏面就保存著釋放的chain。而一般的buf結構體，沒有模組間公用的空閒鏈表池，都是保存在各模組的緩存空閒鏈表池裏面。對於buf結構體，還有一種busy鏈表，表示該鏈表中的buf都處於輸出狀態，如果buf輸出完畢，這些buf就可以釋放並重複利用了。

==========  ========================
功能        函數名
==========  ========================
chain分配   ngx_alloc_chain_link
chain釋放   ngx_free_chain
buf分配     ngx_chain_get_free_buf
buf釋放     ngx_chain_update_chains
==========  ========================


過濾內容的緩存 (90%)
^^^^^^^^^^^^^^^^^^^^^^^^^
由於Nginx設計流式的輸出結構，當我們需要對回應內容作全文過濾的時候，必須緩存部分的buf內容。該類過濾模組往往比較複雜，比如sub，ssi，gzip等模組。這類模組的設計非常靈活，我簡單講一下設計原則：

1. 輸入鏈in需要拷貝操作，經過緩存的過濾模組，輸入輸出鏈往往已經完全不一樣了，所以需要拷貝，通過ngx_chain_add_copy函數完成。

2. 一般有自己的free和busy緩存鏈表池，可以提高buf分配效率。

3. 如果需要分配大塊內容，一般分配固定大小的記憶體卡，並設置recycled標誌，表示可以重複利用。

4. 原有的輸入buf被替換緩存時，必須將其buf->pos設為buf->last，表明原有的buf已經被輸出完畢。或者在新建立的buf，將buf->shadow指向舊的buf，以便輸出完畢時及時釋放舊的buf。


