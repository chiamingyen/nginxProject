其他模組 (40%)
==================
Nginx的模組種類挺多的，除了HTTP模組，還有一些核心模組和mail系列模組。核心模組主要是做一些基礎功能，比如Nginx的啟動初始化，event處理機制，錯誤日誌的初始化，ssl的初始化，正則處理初始化。

mail模組可以對imap，pop3，smtp等協定進行反向代理，這些模組本身不對郵件內容進行處理。

core模組 (40%)
------------------
Nginx的啟動模組 (40%)
+++++++++++++++++++++++++++
啟動模組從啟動Nginx進程開始，做了一系列的初始化工作，源代碼位於src/core/nginx.c，從main函數開始:

1. 時間、正則、錯誤日誌、ssl等初始化

2. 讀入命令行參數

3. OS相關初始化

4. 讀入並解析配置

5. 核心模組初始化

6. 創建各種暫時檔和目錄

7. 創建共用記憶體

8. 打開listen的埠

9. 所有模組初始化

10. 啟動worker進程


event模組 (40%)
--------------------

event的類型和功能 (40%)
+++++++++++++++++++++++++++
Nginx是以event（事件）處理模型為基礎的模組。它為了支援跨平臺，抽象出了event模組。它支援的event處理類型有：AIO（非同步IO），/dev/pool（Solaris 和Unix特有），epoll（Linux特有），eventport（Solaris 10特有），kqueue（BSD特有），pool，rtsig（即時信號），select等。

event模組的主要功能就是，監聽accept後建立的連接，對讀寫事件進行添加刪除。事件處理模型和Nginx的非阻塞IO模型結合在一起使用。當IO可讀可寫的時候，相應的讀寫時間就會被喚醒，此時就會去處理事件的回調函數。

特別對於Linux，Nginx採用的是epoll的EPOLLET（邊沿觸發）的方法來觸發事件，而不是EPOLLLT（水平觸發），所以如果出現了可讀事件，進行處理時，必須讀取所有的可讀數據，否則可能會出現讀事件不再觸發，連接餓死的情況。

.. code-block:: none
		
		typedef struct {
			/* 添加刪除事件 */
			ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
			ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

			ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
			ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
			
			/* 添加刪除連接，會同時監聽讀寫事件 */
			ngx_int_t  (*add_conn)(ngx_connection_t *c);
			ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

			ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
			/* 處理事件的函數 */
			ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
						   ngx_uint_t flags);

			ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
			void       (*done)(ngx_cycle_t *cycle);
		} ngx_event_actions_t;

上述是event處理抽象出來的關鍵結構體，可以看到，每個event處理模型，都需要實現部分功能。最關鍵的是add和del功能，就是最基本的添加和刪除事件的函數。

accept鎖 (40%)
+++++++++++++++++++

Nginx是多進程程式，80埠是各進程所共用的，多進程同時listen 80埠，勢必會產生競爭，也產生了所謂的“驚群”效應。當內核accept一個連接時，會喚醒所有等待中的進程，但實際上只有一個進程能獲取連接，其他的進程都是被無效喚醒的。所以Nginx採用了自有的一套accept加鎖機制，避免多個進程同時調用accept。Nginx多進程的鎖在底層默認是通過CPU自旋鎖來實現。如果作業系統不支援自旋鎖，就採用檔鎖。

Nginx事件處理的入口函數是ngx_process_events_and_timers()，下面是部分代碼，可以看到其加鎖的過程：

.. code-block:: none

		if (ngx_use_accept_mutex) {
			if (ngx_accept_disabled > 0) {
				ngx_accept_disabled--;

			} else {
				if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
					return;
				}
                
				if (ngx_accept_mutex_held) {
					flags |= NGX_POST_EVENTS;

				} else {
					if (timer == NGX_TIMER_INFINITE
						|| timer > ngx_accept_mutex_delay)
					{
						timer = ngx_accept_mutex_delay;
					}
				}
			}
		}

在ngx_trylock_accept_mutex()函數裏面，如果拿到了鎖，Nginx會把listen的埠讀事件加入event處理，該進程在有新連接進來時就可以進行accept了。注意accept操作是一個普通的讀事件。下面的代碼說明了這點：

.. code-block:: none

		(void) ngx_process_events(cycle, timer, flags);

		if (ngx_posted_accept_events) {
			ngx_event_process_posted(cycle, &ngx_posted_accept_events);
		}
		
		if (ngx_accept_mutex_held) {
			ngx_shmtx_unlock(&ngx_accept_mutex);
		}
		
ngx_process_events()函數是所有事件處理的入口，它會遍曆所有的事件。搶到了accept鎖的進程跟一般進程稍微不同的是，它的被加上了NGX_POST_EVENTS標誌，也就是說在ngx_process_events() 函數裏面只接受而不處理事件，並加入post_events的佇列裏面。直到ngx_accept_mutex鎖去掉以後才去處理具體的事件。為什麼這樣？因為ngx_accept_mutex是全局鎖，這樣做可以儘量減少該進程搶到鎖以後，從accept開始到結束的時間，以便其他進程繼續接收新的連接，提高吞吐量。

ngx_posted_accept_events和ngx_posted_events就分別是accept延遲事件佇列和普通延遲事件佇列。可以看到ngx_posted_accept_events還是放到ngx_accept_mutex鎖裏面處理的。該佇列裏面處理的都是accept事件，它會一口氣把內核backlog裏等待的連接都accept進來，註冊到讀寫事件裏。

而ngx_posted_events是普通的延遲事件佇列。一般情況下，什麼樣的事件會放到這個普通延遲佇列裏面呢？我的理解是，那些CPU耗時比較多的都可以放進去。因為Nginx事件處理都是根據觸發順序在一個大循環裏依次處理的，因為Nginx一個進程同時只能處理一個事件，所以有些耗時多的事件會把後面所有的事件處理都被耽擱了。

除了加鎖，Nginx也對各進程的請求處理的均衡性作了優化，也就是說，如果在負載高的時候，進程搶到的鎖過多，會導致這個進程被禁止接受請求一段時間。

比如，在ngx_event_accept函數中，有類似代碼：       

.. code-block:: none

		ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;

ngx_cycle->connection_n是進程可以分配的連接總數，ngx_cycle->free_connection_n是空閒的進程數。上述等式說明了，當前進程的空閒進程數小於1/8的話，就會被禁止accept一段時間。


計時器 (40%)
++++++++++++++++
Nginx在需要用到超時的時候，都會用到計時器機制。比如，建立連接以後的那些讀寫超時。Nginx使用紅黑樹來構造定期器，紅黑樹是一種有序的二叉平衡樹，其查找插入和刪除的複雜度都為O(logn)，所以是一種比較理想的二叉樹。

計時器的機制就是，二叉樹的值是其超時時間，每次查找二叉樹的最小值，如果最小值已經過期，就刪除該節點，然後繼續查找，直到所有超時節點都被刪除。

mail模組
---------------

mail模組的實現
+++++++++++++++

mail模組的功能
+++++++++++++++




