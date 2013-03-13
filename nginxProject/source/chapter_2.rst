nginx平臺初探(30%)
===========================



初探nginx架構(100%)
---------------------
眾所周知，nginx性能高，而nginx的高性能與其架構是分不開的。那麼nginx究竟是怎麼樣的呢？這一節我們先來初識一下nginx框架吧。

nginx在啟動後，在unix系統中會以daemon的方式在後臺運行，後臺進程包含一個master進程和多個worker進程。我們也可以手動地關掉daemon模式，讓nginx在前臺運行，這個時候，nginx就是一個單進程的，很顯然，生產環境下我們肯定不會這麼做，所以關掉daemon的方式，一般是用來調試用的，在後面的章節裏面，我們會詳細地講解如何調試nginx。所以，我們可以看到，nginx是以多進程的方式來工作的，當然nginx也是支援多線程的方式的，只是我們主流的方式還是多進程的方式，也是nginx的默認方式。nginx採用多進程的方式有諸多好處，所以我就主要講解nginx的多進程模式吧。

剛才講到，nginx在啟動後，會有一個master進程和多個worker進程。master進程主要用來管理worker進程，包含：接收來自外界的信號，向各worker進程發送信號，監控worker進程的運行狀態，當worker進程退出後(異常情況下)，會自動重新啟動新的worker進程。而基本的網路事件，則是放在worker進程中來處理了。多個worker進程之間是對等的，他們同等競爭來自用戶端的請求，各進程互相之間是獨立的。一個請求，只可能在一個worker進程中處理，一個worker進程，不可能處理其他進程的請求。worker進程的個數是可以設置的，一般我們會設置與機器cpu核數一致，這裏面的原因與nginx的進程模型以及事件處理模型是分不開的。nginx的進程模型，可以由下圖來表示：

.. image:: /images/chapter-2-1.PNG
    :alt: nginx進程模型
    :align: center

在nginx啟動後，如果我們要操作nginx，要怎麼做呢？從上文中我們可以看到，master來管理worker進程，所以我們只需要與master進程通信就行了。master進程會接收來自外界發來的信號，再根據信號做不同的事情。所以我們要控制nginx，只需要通過kill向master進程發送信號就行了。比如kill -HUP pid，則是告訴nginx，從容地重啟nginx，我們一般用這個信號來重啟nginx，或重新載入配置，因為是從容地重啟，因此服務是不中斷的。master進程在接收到HUP信號後是怎麼做的呢？首先master進程在接到信號後，會先重新載入配置檔，然後再啟動新的進程，並向所有老的進程發送信號，告訴他們可以光榮退休了。新的進程在啟動後，就開始接收新的請求，而老的進程在收到來自master的信號後，就不再接收新的請求，並且在當前進程中的所有未處理完的請求處理完成後，再退出。當然，直接給master進程發送信號，這是比較老的操作方式，nginx在0.8版本之後，引入了一系列命令行參數，來方便我們管理。比如，./nginx -s reload，就是來重啟nginx，./nginx -s stop，就是來停止nginx的運行。如何做到的呢？我們還是拿reload來說，我們看到，執行命令時，我們是啟動一個新的nginx進程，而新的nginx進程在解析到reload參數後，就知道我們的目的是控制nginx來重新載入配置檔了，它會向master進程發送信號，然後接下來的動作，就和我們直接向master進程發送信號一樣了。

現在，我們知道了當我們在操作nginx的時候，nginx內部做了些什麼事情，那麼，worker進程又是如何處理請求的呢？我們前面有提到，worker進程之間是平等的，每個進程，處理請求的機會也是一樣的。當我們提供80埠的http服務時，一個連接請求過來，每個進程都有可能處理這個連接，怎麼做到的呢？首先，每個worker進程都是從master進程fork過來，在master進程裏面，先建立好需要listen的socket之後，然後再fork出多個worker進程，這樣每個worker進程都可以去accept這個socket(當然不是同一個socket，只是每個進程的這個socket會監控在同一個ip位址與埠，這個在網路協定裏面是允許的)。一般來說，當一個連接進來後，所有在accept在這個socket上面的進程，都會收到通知，而只有一個進程可以accept這個連接，其他的則accept失敗，這是所謂的驚群現象。當然，nginx也不會視而不見，所以nginx提供了一個accept_mutex這個東西，從名字上，我們可以看這是一個加在accept上的一把共用鎖。有了這把鎖之後，同一時刻，就只會有一個進程在accpet連接，這樣就不會有驚群問題了。accept_mutex是一個可控選項，我們可以顯示地關掉，默認是打開的。當一個worker進程在accept這個連接之後，就開始讀取請求，解析請求，處理請求，產生資料後，再返回給用戶端，最後才斷開連接，這樣一個完整的請求就是這樣的了。我們可以看到，一個請求，完全由worker進程來處理，而且只在一個worker進程中處理。

那麼，nginx採用這種進程模型有什麼好處呢？當然，好處肯定會很多了。首先，對於每個worker進程來說，獨立的進程，不需要加鎖，所以省掉了鎖帶來的開銷，同時在編程以及問題查上時，也會方便很多。其次，採用獨立的進程，可以讓互相之間不會影響，一個進程退出後，其他進程還在工作，服務不會中斷，master進程則很快重新啟動新的worker進程。當然，worker進程的異常退出，肯定是程式有bug了，異常退出，會導致當前worker上的所有請求失敗，不過不會影響到所有請求，所以降低了風險。當然，好處還有很多，大家可以慢慢體會。

上面講了很多關於nginx的進程模型，接下來，我們來看看nginx的是如何處理事件的。

有人可能要問了，nginx採用多worker的方式來處理請求，每個worker裏面只有一個主線程，那能夠處理的併發數很有限啊，多少個worker就能處理多少個併發，何來高併發呢？非也，這就是nginx的高明之處，nginx採用了非同步非阻塞的方式來處理請求，也就是說，nginx是可以同時處理成千上萬個請求的。想想apache的常用工作方式（apache也有非同步非阻塞版本，但因其與自帶某些模組衝突，所以不常用），每個請求會獨佔一個工作線程，當併發數上到幾千時，就同時有幾千的線程在處理請求了。這對作業系統來說，是個不小的挑戰，線程帶來的記憶體佔用非常大，線程的上下文切換帶來的cpu開銷很大，自然性能就上不去了，而這些開銷完全是沒有意義的。

為什麼nginx可以採用非同步非阻塞的方式來處理呢，或者非同步非阻塞到底是怎麼回事呢？我們先回到原點，看看一個請求的完整過程。首先，請求過來，要建立連接，然後再接收資料，接收資料後，再發送資料。具體到系統底層，就是讀寫事件，而當讀寫事件沒有準備好時，必然不可操作，如果不用非阻塞的方式來調用，那就得阻塞調用了，事件沒有準備好，那就只能等了，等事件準備好了，你再繼續吧。阻塞調用會進入內核等待，cpu就會讓出去給別人用了，對單線程的worker來說，顯然不合適，當網路事件越多時，大家都在等待呢，cpu空閒下來沒人用，cpu利用率自然上不去了，更別談高併發了。好吧，你說加進程數，這跟apache的線程模型有什麼區別，注意，別增加無謂的上下文切換 ？所以，在nginx裏面，最忌諱阻塞的系統調用了。不要阻塞，那就非阻塞嘍。非阻塞就是，事件沒有準備好，馬上返回EAGAIN，告訴你，事件還沒準備好呢，你慌什麼，過會再來吧。好吧，你過一會，再來檢查一下事件，直到事件準備好了為止，在這期間，你就可以先去做其他事情，然後再來看看事件好了沒。雖然不阻塞了，但你得不時地過來檢查一下事件的狀態，你可以做更多的事情了，但帶來的開銷也是不小的。所以，才會有了非同步非阻塞的事件處理機制，具體到系統調用就是像select/poll/epoll/kqueue這樣的系統調用。它們提供了一種機制，讓你可以同時監控多個事件，調用他們是阻塞的，但可以設置超時時間，在超時時間之內，如果有事件準備好了，就返回。這種機制正好解決了我們上面的兩個問題，拿epoll為例(在後面的例子中，我們多以epoll為例子，以代表這一類函數)，當事件沒準備好時，放到epoll裏面，事件準備好了，我們就去讀寫，當讀寫返回EAGAIN時，我們將它再次加入到epoll裏面。這樣，只要有事件準備好了，我們就去處理它，只有當所有事件都沒準備好時，才在epoll裏面等著。這樣，我們就可以併發處理大量的併發了，當然，這裏的併發請求，是指未處理完的請求，線程只有一個，所以同時能處理的請求當然只有一個了，只是在請求間進行不斷地切換而已，切換也是因為非同步事件未準備好，而主動讓出的。這裏的切換是沒有任何代價，你可以理解為迴圈處理多個準備好的事件，事實上就是這樣的。與多線程相比，這種事件處理方式是有很大的優勢的，不需要創建線程，每個請求佔用的記憶體也很少，沒有上下文切換，事件處理非常的羽量級。併發數再多也不會導致無謂的資源浪費（上下文切換）。更多的併發數，只是會佔用更多的記憶體而已。 我之前有對連接數進行過測試，在24G記憶體的機器上，處理的併發請求數達到過200萬。現在的網路服務器基本都採用這種方式，這也是nginx性能高效的主要原因。

我們之前說過，推薦設置worker的個數為cpu的核數，在這裏就很容易理解了，更多的worker數，只會導致進程來競爭cpu資源了，從而帶來不必要的上下文切換。而且，nginx為了更好的利用多核特性，提供了cpu親緣性的綁定選項，我們可以將某一個進程綁定在某一個核上，這樣就不會因為進程的切換帶來cache的失效。像這種小的優化在nginx中非常常見，同時也說明了nginx作者的苦心孤詣。比如，nginx在做4個位元組的字串比較時，會將4個字元轉換成一個int型，再作比較，以減少cpu的指令數等等。

現在，知道了nginx什麼會選擇這樣的進程模型與事件模型了。對於一個基本的web伺服器來說，事件通常有三種類型，網路事件、信號、計時器。從上面的講解中知道，網路事件通過非同步非阻塞可以很好的解決掉。如何處理信號與計時器？

首先，信號的處理。對nginx來說，有一些特定的信號，代表著特定的意義。信號會中斷掉程式當前的運行，在改變狀態後，繼續執行。如果是系統調用，則可能會導致系統調用的失敗，需要重入。關於信號的處理，大家可以學習一些專業書籍，這裏不多說。對於nginx來說，如果nginx正在等待事件（epoll_wait時），如果程式收到信號，在信號處理函數處理完後，epoll_wait會返回錯誤，然後程式可再次進入epoll_wait調用。

另外，再來看看計時器。由於epoll_wait等函數在調用的時候是可以設置一個超時時間的，所以nginx借助這個超時時間來實現計時器。nginx裏面的計時器事件是放在一個最小堆裏面，每次在進入epoll_wait前，先從最小堆裏面拿到所有計時器事件的最小時間，在計算出epoll_wait的超時時間後進入epoll_wait。所以，當沒有事件產生，也沒有中斷信號時，epoll_wait會超時，也就是說，計時器事件到了。這時，nginx會檢查所有的超時事件，將他們的狀態設置為超時，然後再去處理網路事件。由此可以看出，當我們寫nginx代碼時，在處理網路事件的回調函數時，通常做的第一個事情就是判斷超時，然後再去處理網路事件。

我們可以用一段偽代碼來總結一下nginx的事件處理模型：

.. code-block:: none

    while (true) {
        for t in run_tasks:
            t.handler();
        update_time(&now);
        timeout = ETERNITY;
        for t in wait_tasks: /* sorted already */
            if (t.time <= now) {
                t.timeout_handler();
            } else {
                timeout = t.time - now;
                break;
            }
        nevents = poll_function(events, timeout);
        for i in nevents:
            task t;
        if (events[i].type == READ) {
            t.handler = read_handler;
        } else (events[i].type == WRITE) {
            t.handler = write_handler;
        }
        run_tasks_add(t);
    }

好，本節我們講了進程模型，事件模型，包括網路事件，信號，計時器事件。


nginx基礎概念(100%)
---------------------



connection
~~~~~~~~~~~~~~~~~~

在nginx中connection就是對tcp連接的封裝，其中包括連接的socket，讀事件，寫事件。利用nginx封裝的connection，我們可以很方便的使用nginx來處理與連接相關的事情，比如，建立連接，發送與接受資料等。而nginx中的http請求的處理就是建立在connection之上的，所以nginx不僅可以作為一個web伺服器，也可以作為郵件伺服器。當然，利用nginx提供的connection，我們可以與任何後端服務打交道。

結合一個tcp連接的生命週期，我們看看nginx是如何處理一個連接的。首先，nginx在啟動時，會解析配置檔，得到需要監聽的埠與ip位址，然後在nginx的master進程裏面，先初始化好這個監控的socket(創建socket，設置addrreuse等選項，綁定到指定的ip位址埠，再listen)，然後再fork出多個子進程出來，然後子進程會競爭accept新的連接。此時，用戶端就可以向nginx發起連接了。當用戶端與nginx進行三次握手，與nginx建立好一個連接後，此時，某一個子進程會accept成功，得到這個建立好的連接的socket，然後創建nginx對連接的封裝，即ngx_connection_t結構體。接著，設置讀寫事件處理函數並添加讀寫事件來與用戶端進行資料的交換。最後，nginx或用戶端來主動關掉連接，到此，一個連接就壽終正寢了。

當然，nginx也是可以作為用戶端來請求其他server的資料的（如upstream模組），此時，與其他server創建的連接，也封裝在ngx_connection_t中。作為用戶端，nginx先獲取一個ngx_connection_t結構體，然後創建socket，並設置socket的屬性（ 比如非阻塞）。然後再通過添加讀寫事件，調用connect/read/write來調用連接，最後關掉連接，並釋放ngx_connection_t。

在nginx中，每個進程會有一個連接數的最大上限，這個上限與系統對fd的限制不一樣。在作業系統中，通過ulimit -n，我們可以得到一個進程所能夠打開的fd的最大數，即nofile，因為每個socket連接會佔用掉一個fd，所以這也會限制我們進程的最大連接數，當然也會直接影響到我們程式所能支援的最大併發數，當fd用完後，再創建socket時，就會失敗。不過，這裏我要說的nginx對連接數的限制，與nofile沒有直接關係，可以大於nofile，也可以小於nofile。nginx通過設置worker_connectons來設置每個進程可使用的連接最大值。nginx在實現時，是通過一個連接池來管理的，每個worker進程都有一個獨立的連接池，連接池的大小是worker_connections。這裏的連接池裏面保存的其實不是真實的連接，它只是一個worker_connections大小的一個ngx_connection_t結構的陣列。並且，nginx會通過一個鏈表free_connections來保存所有的空閒ngx_connection_t，每次獲取一個連接時，就從空閒連接鏈表中獲取一個，用完後，再放回空閒連接鏈表裏面。

在這裏，很多人會誤解worker_connections這個參數的意思，認為這個值就是nginx所能建立連接的最大值。其實不然，這個值是表示每個worker進程所能建立連接的最大值，所以，一個nginx能建立的最大連接數，應該是worker_connections * worker_processes。當然，這裏說的是最大連接數，對於HTTP請求本地資源來說，能夠支持的最大併發數量是worker_connections * worker_processes，而如果是HTTP作為反向代理來說，最大併發數量應該是worker_connections * worker_processes/2。因為作為反向代理伺服器，每個併發會建立與用戶端的連接和與後端服務的連接，會佔用兩個連接。

那麼，我們前面有說過一個用戶端連接過來後，多個空閒的進程，會競爭這個連接，很容易看到，這種競爭會導致不公平，如果某個進程得到accept的機會比較多，它的空閒連接很快就用完了，如果不提前做一些控制，當accept到一個新的tcp連接後，因為無法得到空閒連接，而且無法將此連接轉交給其他進程，最終會導致此tcp連接得不到處理，就中止掉了。很顯然，這是不公平的，有的進程有空餘連接，卻沒有處理機會，有的進程因為沒有空餘連接，卻人為地丟棄連接。那麼，如何解決這個問題呢？首先，nginx的處理得先打開accept_mutex選項，此時，只有獲得了accept_mutex的進程才會去添加accept事件，也就是說，nginx會控制進程是否添加accept事件。nginx使用一個叫ngx_accept_disabled的變數來控制是否去競爭accept_mutex鎖。在第一段代碼中，計算ngx_accept_disabled的值，這個值是nginx單進程的所有連接總數的八分之一，減去剩下的空閒連接數量，得到的這個ngx_accept_disabled有一個規律，當剩餘連接數小於總連接數的八分之一時，其值才大於0，而且剩餘的連接數越小，這個值越大。再看第二段代碼，當ngx_accept_disabled大於0時，不會去嘗試獲取accept_mutex鎖，並且將ngx_accept_disabled減1，於是，每次執行到此處時，都會去減1，直到小於0。不去獲取accept_mutex鎖，就是等於讓出獲取連接的機會，很顯然可以看出，當空餘連接越少時，ngx_accept_disable越大，於是讓出的機會就越多，這樣其他進程獲取鎖的機會也就越大。不去accept，自己的連接就控制下來了，其他進程的連接池就會得到利用，這樣，nginx就控制了多進程間連接的平衡了。

.. code-block:: none

    ngx_accept_disabled = ngx_cycle->connection_n / 8
        - ngx_cycle->free_connection_n;

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

好了，連接就先介紹到這，本章的目的是介紹基本概念，知道在nginx中連接是個什麼東西就行了，而且連接是屬於比較高級的用法，在後面的模組開發高級篇會有專門的章節來講解連接與事件的實現及使用。



request
~~~~~~~~~~~~~~~~~~

這節我們講request，在nginx中我們指的是http請求，具體到nginx中的資料結構是ngx_http_request_t。ngx_http_request_t是對一個http請求的封裝。 我們知道，一個http請求，包含請求行、請求頭、請求體、回應行、回應頭、回應體。

http請求是典型的請求-回應類型的的網路協定，而http是檔協議，所以我們在分析請求行與請求頭，以及輸出回應行與回應頭，往往是一行一行的進行處理。如果我們自己來寫一個http伺服器，通常在一個連接建立好後，用戶端會發送請求過來。然後我們讀取一行資料，分析出請求行中包含的method、uri、http_version資訊。然後再一行一行處理請求頭，並根據請求method與請求頭的資訊來決定是否有請求體以及請求體的長度，然後再去讀取請求體。得到請求後，我們處理請求產生需要輸出的資料，然後再生成回應行，回應頭以及回應體。在將回應發送給用戶端之後，一個完整的請求就處理完了。當然這是最簡單的webserver的處理方式，其實nginx也是這樣做的，只是有一些小小的區別，比如，當請求頭讀取完成後，就開始進行請求的處理了。nginx通過ngx_http_request_t來保存解析請求與輸出回應相關的資料。

那接下來，簡要講講nginx是如何處理一個完整的請求的。對於nginx來說，一個請求是從ngx_http_init_request開始的，在這個函數中，會設置讀事件為ngx_http_process_request_line，也就是說，接下來的網路事件，會由ngx_http_process_request_line來執行。從ngx_http_process_request_line的函數名，我們可以看到，這就是來處理請求行的，正好與之前講的，處理請求的第一件事就是處理請求行是一致的。通過ngx_http_read_request_header來讀取請求資料。然後調用ngx_http_parse_request_line函數來解析請求行。nginx為提高效率，採用狀態機來解析請求行，而且在進行method的比較時，沒有直接使用字串比較，而是將四個字元轉換成一個整型，然後一次比較以減少cpu的指令數，這個前面有說過。很多人可能很清楚一個請求行包含請求的方法，uri，版本，卻不知道其實在請求行中，也是可以包含有host的。比如一個請求GET    http://www.taobao.com/uri HTTP/1.0這樣一個請求行也是合法的，而且host是www.taobao.com，這個時候，nginx會忽略請求頭中的host域，而以請求行中的這個為准來查找虛擬主機。另外，對於對於http0.9版來說，是不支持請求頭的，所以這裏也是要特別的處理。所以，在後面解析請求頭時，協議版本都是1.0或1.1。整個請求行解析到的參數，會保存到ngx_http_request_t結構當中。

在解析完請求行後，nginx會設置讀事件的handler為ngx_http_process_request_headers，然後後續的請求就在ngx_http_process_request_headers中進行讀取與解析。ngx_http_process_request_headers函數用來讀取請求頭，跟請求行一樣，還是調用ngx_http_read_request_header來讀取請求頭，調用ngx_http_parse_header_line來解析一行請求頭，解析到的請求頭會保存到ngx_http_request_t的域headers_in中，headers_in是一個鏈表結構，保存所有的請求頭。而HTTP中有些請求是需要特別處理的，這些請求頭與請求處理函數存放在一個映射表裏面，即ngx_http_headers_in，在初始化時，會生成一個hash表，當每解析到一個請求頭後，就會先在這個hash表中查找，如果有找到，則調用相應的處理函數來處理這個請求頭。比如:Host頭的處理函數是ngx_http_process_host。

當nginx解析到兩個回車換行符時，就表示請求頭的結束，此時就會調用ngx_http_process_request來處理請求了。ngx_http_process_request會設置當前的連接的讀寫事件處理函數為ngx_http_request_handler，然後再調用ngx_http_handler來真正開始處理一個完整的http請求。這裏可能比較奇怪，讀寫事件處理函數都是ngx_http_request_handler，其實在這個函數中，會根據當前事件是讀事件還是寫事件，分別調用ngx_http_request_t中的read_event_handler或者是write_event_handler。由於此時，我們的請求頭已經讀取完成了，之前有說過，nginx的做法是先不讀取請求body，所以這裏面我們設置read_event_handler為ngx_http_block_reading，即不讀取資料了。剛才說到，真正開始處理資料，是在ngx_http_handler這個函數裏面，這個函數會設置write_event_handler為ngx_http_core_run_phases，並執行ngx_http_core_run_phases函數。ngx_http_core_run_phases這個函數將執行多階段請求處理，nginx將一個http請求的處理分為多個階段，那麼這個函數就是執行這些階段來產生資料。因為ngx_http_core_run_phases最後會產生資料，所以我們就很容易理解，為什麼設置寫事件的處理函數為ngx_http_core_run_phases了。在這裏，我簡要說明了一下函數的調用邏輯，我們需要明白最終是調用ngx_http_core_run_phases來處理請求，產生的回應頭會放在ngx_http_request_t的headers_out中，這一部分內容，我會放在請求處理流程裏面去講。nginx的各種階段會對請求進行處理，最後會調用filter來過濾資料，對資料進行加工，如truncked傳輸、gzip壓縮等。這裏的filter包括header filter與body filter，即對回應頭或回應體進行處理。filter是一個鏈表結構，分別有header filter與body filter，先執行header filter中的所有filter，然後再執行body filter中的所有filter。在header filter中的最後一個filter，即ngx_http_header_filter，這個filter將會遍曆所的有回應頭，最後需要輸出的回應頭在一個連續的記憶體，然後調用ngx_http_write_filter進行輸出。ngx_http_write_filter是body filter中的最後一個，所以nginx首先的body資訊，在經過一系列的body filter之後，最後也會調用ngx_http_write_filter來進行輸出(有圖來說明)。

這裏要注意的是，nginx會將整個請求頭都放在一個buffer裏面，這個buffer的大小通過配置項client_header_buffer_size來設置，如果用戶的請求頭太大，這個buffer裝不下，那nginx就會重新分配一個新的更大的buffer來裝請求頭，這個大buffer可以通過large_client_header_buffers來設置，這個large_buffer這一組buffer，比如配置4 8k，就是表示有四個8k大小的buffer可以用。注意，為了保存請求行或請求頭的完整性，一個完整的請求行或請求頭，需要放在一個連續的記憶體裏面，所以，一個完整的請求行或請求頭，只會保存在一個buffer裏面。這樣，如果請求行大於一個buffer的大小，就會返回414錯誤，如果一個請求頭大小大於一個buffer大小，就會返回400錯誤。在瞭解了這些參數的值，以及nginx實際的做法之後，在應用場景，我們就需要根據實際的需求來調整這些參數，來優化我們的程式了。

處理流程圖：

.. image:: /images/chapter-2-2.PNG
    :alt: 請求處理流程
    :align: center

以上這些，就是nginx中一個http請求的生命週期了。我們再看看與請求相關的一些概念吧。

keepalive
^^^^^^^^^^^^^^^^^
當然，在nginx中，對於http1.0與http1.1也是支持長連接的。什麼是長連接呢？我們知道，http請求是某於TCP協議之上的，那麼，當用戶端在發起請求前，需要先與服務端建立TCP連接，而每一次的TCP連接是需要三次握手來確定的，如果用戶端與服務端之間網路差一點，這三次交互消費的時間會比較多，而且三次交互也會帶來網路流量。當然，當連接斷開後，也會有四次的交互，當然對用戶體驗來說就不重要了。而http請求是請求應答式的，如果我們能知道每個請求頭與響應體的長度，那麼我們是可以在一個連接上面執行多個請求的，這就是所謂的長連接，但前提條件是我們先得確定請求頭與回應體的長度。對於請求來說，如果當前請求需要有body，如POST請求，那麼nginx就需要用戶端在請求頭中指定content-length來表明body的大小，否則返回400錯誤。也就是說，請求體的長度是確定的，那麼回應體的長度呢？先來看看http協議中關於回應body長度的確定：

1. 對於http1.0協議來說，如果回應頭中有content-length頭，則以content-length的長度就可以知道body的長度了，用戶端在接收body時，就可以依照這個長度來接收資料，接收完後，就表示這個請求完成了。而如果沒有content-length頭，則用戶端會一直接收資料，直到服務端主動斷開連接，才表示body接收完了。

2. 而對於http1.1協議來說，如果回應頭中的Transfer-encoding為chunked傳輸，則表示body是流式輸出，body會被分成多個塊，每塊的開始會標識出當前塊的長度，此時，body不需要通過長度來指定。如果是非chunked傳輸，而且有content-length，則按照content-length來接收資料。否則，如果是非chunked，並且沒有content-length，則用戶端接收資料，直到服務端主動斷開連接。

從上面，我們可以看到，除了http1.0不帶content-length以及http1.1非chunked不帶content-length外，body的長度是可知的。此時，當服務端在輸出完body之後，會可以考慮使用長連接。能否使用長連接，也是有條件限制的。如果用戶端的請求頭中的connection為close，則表示用戶端需要關掉長連接，如果為keep-alive，則用戶端需要打開長連接，如果用戶端的請求中沒有connection這個頭，那麼根據協定，如果是http1.0，則默認為close，如果是http1.1，則默認為keep-alive。如果的結果為keepalive，那麼，nginx在輸出完回應體後，會設置當前連接的keepalive屬性，然後再次等待用戶端的下一次請求資料。當然，nginx不可能一直等待下去，如果用戶端一直不發資料過來，豈不是一直佔用這個連接？所以當nginx直接keepalive等待下一次的請求時，會有一個最大等待時間，而這個時間是通過選項keepalive_timeout來配置的，如果配置為0，則表示關掉keepalive，此時，http版本無論是1.1還是1.0，用戶端的connection不管是close還是keepalive，都會強制為close。

如果服務端最後決定的是keepalive打開，那麼在回應的http頭裏面，也會包含有connection，其值是"Keep-Alive"，否則就是"Close"。如果connection值為close，那麼在nginx回應完資料後，會主動關掉連接。所以，對於請求量比較大的nginx來說，關掉keepalive最後會產生比較多的time-wait狀態的socket。一般來說，當用戶端的一次訪問，需要多次訪問同一個server時，打開keepalive的優勢非常大，比如圖片伺服器，通常一個網頁會包含很多個圖片。打開keepalive也會大量減少time-wait的數量。

pipe
^^^^^^^^^^^^^^^^^
在http1.1中，引入了一種新的特性，即pipeline。那麼什麼是pipeling呢？pipeling其實就是流水線作業，它可以看作為keepalive的一種昇華，因為pipeling也是基於長連接的，目的就是利用一個連接作多次請求。對之前的keepalive來說，如果用戶端要提交多個請求，那麼第二個請求，必須要等到第一個請求的回應接收完全後，才能發起，也就是說，請求是串列進行的，一個請求接一個請求。注意，一個完整的請求，包括發送請求，處理請求，回應請求。而對pipeline來說，用戶端不必等到第一個請求處理完後，就可以馬上發起第二個請求。我們知道，tcp連接是全雙工的，發送與接收可以同時進行，所以，我們可以將多個請求頭依次發送出去，在服務端依次處理，用戶端再依次接收，這樣就多個請求就是同時進行的了。nginx是直接支持pipeling的，但是，nginx對pipeling中的多個請求的處理卻不是並行的，依然是一個請求接一個請求的處理，只是在處理第一個請求的時候，用戶端就可以發起第二個請求。這樣，nginx利用pipeline減少了處理完一個請求後，等待第二個請求的請求頭資料的時間。其實nginx的做法很簡單，前面說到，nginx在讀取資料時，會將讀取的資料放到一個buffer裏面，所以，如果nginx在處理完前一個請求後，如果發現buffer裏面還有資料，就認為剩下的資料是下一個請求的開始，然後就接下來處理下一個請求，否則就設置keepalive。

lingering_close
^^^^^^^^^^^^^^^^^
lingering_close，字面意思就是延遲關閉，也就是說，當nginx要關閉連接時，並非立即關閉連接，而是再等待一段時間後才真正關掉連接。為什麼要這樣呢？我們先來看看這樣一個場景。nginx在接收用戶端的請求時，可能由於用戶端或服務端出錯了，要立即回應錯誤資訊給用戶端，而nginx在回應錯誤資訊後，大分部情況下是需要關閉當前連接。如果客戶端正在發送資料，或資料還沒有到達服務端，服務端就將連接關掉了。那麼，用戶端發送的資料會收到RST包，此時，用戶端對於接收到的服務端的資料，將不會發送ACK，也就是說，用戶端將不會拿到服務端發送過來的錯誤資訊資料。那用戶端肯定會想，這伺服器好霸道，動不動就reset我的連接，連個錯誤資訊都沒有。

在上面這個場景中，我們可以看到，關鍵點是服務端給用戶端發送了RST包，導致自己發送的資料在用戶端忽略掉了。所以，解決問題的重點是，讓服務端別發RST包。再想想，我們發送RST是因為我們關掉了連接，關掉連接是因為我們不想再處理此連接了，也不會有任何資料產生了。對於全雙工的TCP連接來說，我們只需要關掉寫就行了，讀可以繼續進行，我們只需要丟掉讀到的任何資料就行了，這樣的話，當我們關掉連接後，用戶端再發過來的資料，就不會再收到RST了。當然最終我們還是需要關掉這個讀端的，所以我們會設置一個超時時間，在這個時間過後，就關掉讀，用戶端再發送資料來就不管了，作為服務端我會認為，都這麼長時間了，發給你的錯誤資訊也應該讀到了，再慢就不關我事了，要怪就怪你RP不好了。當然，正常的用戶端，在讀取到資料後，會關掉連接，此時服務端就會在超時時間內關掉讀端。這些正是lingering_close所做的事情。協定棧提供 SO_LINGER 這個選項，它的一種配置情況就是來處理lingering_close的情況的，不過nginx是自己實現的lingering_close。lingering_close存在的意義就是來讀取剩下的用戶端發來的資料，所以nginx會有一個讀超時時間，通過lingering_timeout選項來設置，如果在lingering_timeout時間內還沒有收到資料，則直接關掉連接。nginx還支持設置一個總的讀取時間，通過lingering_time來設置，這個時間也就是nginx在關閉寫之後，保留socket的時間，用戶端需要在這個時間內發送完所有的資料，否則nginx在這個時間過後，會直接關掉連接。當然，nginx是支援配置是否打開lingering_close選項的，通過lingering_close選項來配置。
那麼，我們在實際應用中，是否應該打開lingering_close呢？這個就沒有固定的推薦值了，如Maxim Dounin所說，lingering_close的主要作用是保持更好的用戶端相容性，但是卻需要消耗更多的額外資源（比如連接會一直占著）。

這節，我們介紹了nginx中，連接與請求的基本概念，下節，我們講基本的資料結構。


基本資料結構(20%)
----------------------
nginx的作者為追求極致的高效，自己實現了很多頗具特色的nginx風格的資料結構以及公共函數。比如，nginx提供了帶長度的字串，根據編譯器選項優化過的字串拷貝函數ngx_copy等。所以，在我們寫nginx模組時，應該儘量調用nginx提供的api，儘管有些api只是對glibc的巨集定義。本節，我們介紹string、list、buffer、chain等一系列最基本的資料結構及相關api的使用技巧以及注意事項。


ngx_str_t(100%)
~~~~~~~~~~~~~~~~~~
在nginx源碼目錄的src/core下麵的ngx_string.h|c裏面，包含了字串的封裝以及字串相關操作的api。nginx提供了一個帶長度的字串結構ngx_str_t，它的原型如下：

.. code-block:: none

    typedef struct {
        size_t      len;
        u_char     *data;
    } ngx_str_t;

從結構體當中，data指向字串資料的第一個字元，字串的結束用長度來表示，而不是由'\0'來表示結束。所以，在寫nginx代碼時，處理字串的方法跟我們平時使用有很大的不一樣，但要時刻記住，字串不以'\0'結束，儘量使用nginx提供的字串操作的api來操作字串。
那麼，nginx這樣做有什麼好處呢？首先，通過長度來表示字串長度，減少計算字串長度的次數。其次，nginx可以重複引用一段字串記憶體，data可以指向任意記憶體，長度表示結束，而不用去copy一份自己的字串(因為如果要以\0結束，而不能更改原字串，所以勢必要copy一段字串)。我們在ngx_http_request_t結構體的成員中，可以找到很多字串引用一段記憶體的例子，比如request_line、uri、args等等，這些字串的data部分，都是指向在接收資料時創建buffer所指向的記憶體中，uri，args就沒有必要copy一份出來。這樣的話，減少了很多不必要的記憶體分配與拷貝。
正是由於有這樣的特性，當你在修改一個字串時，你就得注意，你修改的字串是否可以被修改，如果修改後，是否會對其他引用產生影響。在後面介紹ngx_unescape_uri函數的時候，就會看到這一點。然後，使用nginx的字串會產生一些問題，glibc提供的很多系統api函數大多是通過'\0'來表示字串的結束，所以我們在調用系統api時，就不能直接傳入str->data了。此時，通常的做法是創建一段str->len + 1大小的記憶體，然後copy字串，最後一個位元組置為'\0'。比較hack的做法是，將字串最後一個字元的後一個字元backup一個，然後設置為'\0'，在做完調用後，再由backup改回來，但前提條件是，你得確定這個字元是可以修改的，而且是有記憶體分配，不會越界，但一般不建議這麼做。
接下來，看看nginx提供的操作字串相關的api。


.. code-block:: none

    ngx_string(str)

初始化一個字串為str，str必須為常量字串，  一般只用於聲明字串變數時順便初始化變數的值。

.. code-block:: none

    ngx_null_string

聲明變數時，初始化字串為空字串，符串的長度為0，data為NULL。

.. code-block:: none

    ngx_str_set(str, text)

設置字串str為text，text必須為常量字串。

.. code-block:: none

    ngx_str_null(str) 

設置字串str為空串，長度為0，data為NULL。

上面這四個函數，使用時一定要小心，ngx_string與ngx_null_string只能用於賦值時初始化，如：

.. code-block:: none

    ngx_str_t str = ngx_string("hello world");
    ngx_str_t str1 = ngx_null_string();

如果這樣使用，就會有問題：


.. code-block:: none

    ngx_str_t str, str1;
    str = ngx_string("hello world");    // 編譯出錯
    str1 = ngx_null_string();                // 編譯出錯

這種情況，可以調用ngx_str_set與ngx_str_null這兩個函數來做:

.. code-block:: none

    ngx_str_t str, str1;
    ngx_str_set(str, "hello world");    
    ngx_str_null(str);

不過要注意的是，ngx_string與ngx_str_set在調用時，傳進去的字串一定是常量字串，否則會得到意想不到的錯誤。如： 

.. code-block:: none

   ngx_str_t str;
   u_char *a = "hello world";
   ngx_str_set(str, a);    // 問題產生


.. code-block:: none

   void ngx_strlow(u_char *dst, u_char *src, size_t n);

將src的前n個字元轉換成小寫存放在dst字串當中，調用者需要保證dst指向的空間大於等於n。操作不會對原字串產生變動。如要更改原字串，可以：

.. code-block:: none

    ngx_str_t str = ngx_string("hello world");
    ngx_strlow(str->data, str->data, str->len);


.. code-block:: none

    ngx_strncmp(s1, s2, n)

不區分大小寫的字串比較，只比較前n個字元。


.. code-block:: none

    ngx_strcmp(s1, s2)

不區分大小寫的不帶長度的字串比較。

.. code-block:: none

    ngx_int_t ngx_strcasecmp(u_char *s1, u_char *s2);

區分大小寫的不帶長度的字串比較。

.. code-block:: none

    ngx_int_t ngx_strncasecmp(u_char *s1, u_char *s2, size_t n);

區分大小寫的帶長度的字串比較，只比較前n個字元。

.. code-block:: none

    u_char * ngx_cdecl ngx_sprintf(u_char *buf, const char *fmt, ...);
    u_char * ngx_cdecl ngx_snprintf(u_char *buf, size_t max, const char *fmt, ...);
    u_char * ngx_cdecl ngx_slprintf(u_char *buf, u_char *last, const char *fmt, ...);

上面這三個函數用於字串格式化，ngx_snprintf的第二個參數max指明buf的空間大小，ngx_slprintf則通過last來指明buf空間的大小。推薦使用第二個或第三個函數來格式化字串，ngx_sprintf函數還是比較危險的，容易產生緩衝區溢出漏洞。在這一系列函數中，nginx在相容glibc中格式化字串的形式之外，還添加了一些方便格式化nginx類型的一些轉義字元，比如%V用於格式化ngx_str_t結構。在nginx原始檔案的ngx_string.c中有說明：

.. code-block:: none

    /*
     * supported formats:
     *    %[0][width][x][X]O        off_t
     *    %[0][width]T              time_t
     *    %[0][width][u][x|X]z      ssize_t/size_t
     *    %[0][width][u][x|X]d      int/u_int
     *    %[0][width][u][x|X]l      long
     *    %[0][width|m][u][x|X]i    ngx_int_t/ngx_uint_t
     *    %[0][width][u][x|X]D      int32_t/uint32_t
     *    %[0][width][u][x|X]L      int64_t/uint64_t
     *    %[0][width|m][u][x|X]A    ngx_atomic_int_t/ngx_atomic_uint_t
     *    %[0][width][.width]f      double, max valid number fits to %18.15f
     *    %P                        ngx_pid_t
     *    %M                        ngx_msec_t
     *    %r                        rlim_t
     *    %p                        void *
     *    %V                        ngx_str_t *
     *    %v                        ngx_variable_value_t *
     *    %s                        null-terminated string
     *    %*s                       length and string
     *    %Z                        '\0'
     *    %N                        '\n'
     *    %c                        char
     *    %%                        %
     *
     *  reserved:
     *    %t                        ptrdiff_t
     *    %S                        null-terminated wchar string
     *    %C                        wchar
     */

這裏特別要提醒的是，我們最常用於格式化ngx_str_t結構，其對應的轉義符是%V，傳給函數的一定要是指標類型，否則程式就會coredump掉。這也是我們最容易犯的錯。比如：

.. code-block:: none

    ngx_str_t str = ngx_string("hello world");
    char buffer[1024];
    ngx_snprintf(buffer, 1024, "%V", &str);    // 注意，str取地址

.. code-block:: none

    void ngx_encode_base64(ngx_str_t *dst, ngx_str_t *src);
    ngx_int_t ngx_decode_base64(ngx_str_t *dst, ngx_str_t *src);

這兩個函數用於對str進行base64編碼與解碼，調用前，需要保證dst中有足夠的空間來存放結果，如果不知道具體大小，可先調用ngx_base64_encoded_length與ngx_base64_decoded_length來預估最大佔用空間。

.. code-block:: none

    uintptr_t ngx_escape_uri(u_char *dst, u_char *src, size_t size,
        ngx_uint_t type);

對src進行編碼，根據type來按不同的方式進行編碼，如果dst為NULL，則返回需要轉義的字元的數量，由此可得到需要的空間大小。type的類型可以是：

.. code-block:: none

    #define NGX_ESCAPE_URI         0
    #define NGX_ESCAPE_ARGS        1
    #define NGX_ESCAPE_HTML        2
    #define NGX_ESCAPE_REFRESH     3
    #define NGX_ESCAPE_MEMCACHED   4
    #define NGX_ESCAPE_MAIL_AUTH   5

.. code-block:: none

    void ngx_unescape_uri(u_char **dst, u_char **src, size_t size, ngx_uint_t type);

對src進行反編碼，type可以是0、NGX_UNESCAPE_URI、NGX_UNESCAPE_REDIRECT這三個值。如果是0，則表示src中的所有字元都要進行轉碼。如果是NGX_UNESCAPE_URI與NGX_UNESCAPE_REDIRECT，則遇到'?'後就結束了，後面的字元就不管了。而NGX_UNESCAPE_URI與NGX_UNESCAPE_REDIRECT之間的區別是NGX_UNESCAPE_URI對於遇到的需要轉碼的字元，都會轉碼，而NGX_UNESCAPE_REDIRECT則只會對非可見字元進行轉碼。

.. code-block:: none

    uintptr_t ngx_escape_html(u_char *dst, u_char *src, size_t size);

對html標籤進行編碼。

當然，我這裏只介紹了一些常用的api的使用，大家可以先熟悉一下，在實際使用過程中，遇到不明白的，最快最直接的方法就是去看源碼，看api的實現或看nginx自身調用api的地方是怎麼做的，代碼就是最好的文檔。

ngx_pool_t(100%)
~~~~~~~~~~~~~~~~~~

ngx_pool_t是一個非常重要的資料結構，在很多重要的場合都有使用，很多重要的資料結構也都在使用它。那麼它究竟是一個什麼東西呢？簡單的說，它提供了一種機制，幫助管理一系列的資源（如記憶體，檔等），使得對這些資源的使用和釋放統一進行，免除了使用過程中考慮到對各種各樣資源的什麼時候釋放，是否遺漏了釋放的擔心。

例如對於記憶體的管理，如果我們需要使用記憶體，那麼總是從一個ngx_pool_t的物件中獲取記憶體，在最終的某個時刻，我們銷毀這個ngx_pool_t物件，所有這些記憶體都被釋放了。這樣我們就不必要對對這些記憶體進行malloc和free的操作，不用擔心是否某塊被malloc出來的記憶體沒有被釋放。因為當ngx_pool_t物件對銷毀的時候，所有從這個物件中分配出來的記憶體都會被統一釋放掉。

在比如我們要使用一系列的檔，但是我們打開以後，最終需要都關閉，那麼我們就把這些檔統一登記到一個ngx_pool_t物件中，當這個ngx_pool_t物件被銷毀的時候，所有這些檔都將會被關閉。

從上面舉的兩個例子中我們可以看出，使用ngx_pool_t這個資料結構的時候，所有的資源的釋放都在這個物件被銷毀的時刻，統一進行了釋放，那麼就會帶來一個問題，就是這些資源的生存週期（或者說被佔用的時間）是跟ngx_pool_t的生存週期基本一致（ngx_pool_t也提供了少量操作可以提前釋放資源）。從最高效的角度來說，這並不是最好的。比如，我們需要依次使用A，B，C三個資源，且使用完B的時候，A就不會再被使用了，使用C的時候A和B都不會被使用到。如果不使用ngx_pool_t來管理這三個資源，那我們可能從系統裏面申請A，使用A，然後在釋放A。接著申請B，使用B，再釋放B。最後申請C，使用C，然後釋放C。但是當我們使用一個ngx_pool_t物件來管理這三個資源的時候，A，B和C的是否是在最後一起發生的，也就是在使用完C以後。誠然，這在客觀上增加了程式在一段時間的資源使用量。但是這也減輕了程式師分別管理三個資源的生命週期的工作。這也就是有所得，必有所失的道理。實際上是一個取捨的問題，在具體的情況下，你更在乎的是哪個。

可以看一下在nginx裏面一個典型的使用ngx_pool_t的場景，對於nginx處理的每個http request, nginx會生成一個ngx_pool_t物件與這個http requst關聯，所有處理過程中需要申請的資源都從這個ngx_pool_t物件中獲取，當這個http requst處理完成以後，所有在處理過程中申請的資源，都講隨著這個關聯的ngx_pool_t物件的銷毀而釋放。

ngx_pool_t相關結構及操作被定義在檔src/core/ngx_palloc.h|c中。

.. code-block:: none 

    typedef struct ngx_pool_s        ngx_pool_t; 

    struct ngx_pool_s {
        ngx_pool_data_t       d;
        size_t                max;
        ngx_pool_t           *current;
        ngx_chain_t          *chain;
        ngx_pool_large_t     *large;
        ngx_pool_cleanup_t   *cleanup;
        ngx_log_t            *log;
    };


從ngx_pool_t的一般使用者的角度來說，可不用關注ngx_pool_t結構中各欄位作用。所以這裏也不會進行詳細的解釋，當然在說明某些操作函數的使用的時候，如有必要，會進行說明。

下面我們來分別解釋下ngx_pool_t的相關操作。

.. code-block:: none  

    ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
                                                              
                                                              
創建一個初始節點大小為size的pool，log為後續在該pool上進行操作時輸出日誌的物件。 需要說明的是size的選擇，size的大小必須小於等於NGX_MAX_ALLOC_FROM_POOL，且必須大於sizeof(ngx_pool_t)。 

選擇大於NGX_MAX_ALLOC_FROM_POOL的值會造成浪費，因為大於該限制的空間不會被用到（只是說在第一個由ngx_pool_t物件管理的記憶體塊上的記憶體，後續的分配如果第一個記憶體塊上的空閒部分已用完，會再分配的）。 

選擇小於sizeof(ngx_pool_t)的值會造成程式奔潰。由於初始大小的記憶體塊中要用一部分來存儲ngx_pool_t這個資訊本身。

當一個ngx_pool_t物件被創建以後，改物件的max欄位被賦值為size-sizeof(ngx_pool_t)和NGX_MAX_ALLOC_FROM_POOL這兩者中比較小的。後續的從這個pool中分配的記憶體塊，在第一塊記憶體使用完成以後，如果要繼續分配的話，就需要繼續從作業系統申請記憶體。當記憶體的大小小於等於max欄位的時候，則分配新的記憶體塊，鏈結在d這個欄位（實際上是d.next欄位）管理的一條鏈表上。當要分配的記憶體塊是比max大的，那麼從系統中申請的記憶體是被掛接在large欄位管理的一條鏈表上。我們暫且把這個稱之為大塊記憶體鏈和小塊記憶體鏈。


.. code-block:: none   

    void *ngx_palloc(ngx_pool_t *pool, size_t size); 

從這個pool中分配一塊為size大小的記憶體。注意，此函數分配的記憶體的起始位址按照NGX_ALIGNMENT進行了對齊。對齊操作會提高系統處理的速度，但會造成少量記憶體的浪費。 


.. code-block:: none   

    void *ngx_pnalloc(ngx_pool_t *pool, size_t size); 

從這個pool中分配一塊為size大小的記憶體。但是此函數分配的記憶體並沒有像上面的函數那樣進行過對齊。


.. code-block:: none

    void *ngx_pcalloc(ngx_pool_t *pool, size_t size);

該函數也是分配size大小的記憶體，並且對分配的記憶體塊進行了清零。內部實際上是轉調用ngx_palloc實現的。 


.. code-block:: none

    void *ngx_prealloc(ngx_pool_t *pool, void *p, size_t old_size, size_t new_size);

對指標p指向的一塊記憶體再分配。如果p是NULL，則直接分配一塊新的new_size大小的記憶體。 

如果p不是NULL, 新分配一塊記憶體，並把舊記憶體中的內容拷貝至新記憶體塊中，然後釋放p的舊記憶體（具體能不能釋放舊的，要視具體的情況而定，這裏不再詳述）。

這個函數實際上也是使用ngx_palloc實現的。


.. code-block:: none 

    void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);

按照指定對齊大小alignment來申請一塊大小為size的記憶體。此處獲取的記憶體不管大小都將被置於大記憶體塊鏈中管理。 


.. code-block:: none  

    ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);

對於被置於大塊記憶體鏈，也就是被large欄位管理的一列記憶體中的某塊進行釋放。該函數的實現是順序遍曆large管理的大塊記憶體鏈表。所以效率比較低下。如果在這個鏈表中找到了這塊記憶體，則釋放，並返回NGX_OK。否則返回NGX_DECLINED。

由於這個操作效率比較低下，除非必要，也就是說這塊記憶體非常大，確應及時釋放，否則一般不需要調用。反正記憶體在這個pool被銷毀的時候，總歸會都釋放掉的嘛！


.. code-block:: none 

    ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size); 

ngx_pool_t中的cleanup欄位管理著一個特殊的鏈表，該鏈表的每一項都記錄著一個特殊的需要釋放的資源。對於這個鏈表中每個節點所包含的資源如何去釋放，是自說明的。這也就提供了非常大的靈活性。意味著，ngx_pool_t不僅僅可以管理記憶體，通過這個機制，也可以管理任何需要釋放的資源，例如，關閉檔，或者刪除檔等等的。下面我們看一下這個鏈表每個節點的類型: 

.. code-block:: none  

    typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;
    typedef void (*ngx_pool_cleanup_pt)(void *data);

    struct ngx_pool_cleanup_s {
        ngx_pool_cleanup_pt   handler;
        void                 *data;
        ngx_pool_cleanup_t   *next;
    };

:data: 指明了該節點所對應的資源。 

:handler: 是一個函數指標，指向一個可以釋放data所對應資源的函數。該函數的只有一個參數，就是data。 

:next: 指向該鏈表中下一個元素。

看到這裏，ngx_pool_cleanup_add這個函數的用法，我相信大家都應該有一些明白了。但是這個參數size是起什麼作用的呢？這個 size就是要存儲這個data欄位所指向的資源的大小。

比如我們需要最後刪除一個檔。那我們在調用這個函數的時候，把size指定為存儲檔案名的字串的大小，然後調用這個函數給cleanup鏈表中增加一項。該函數會返回新添加的這個節點。我們然後把這個節點中的data欄位拷貝為檔案名。把hander欄位賦值為一個刪除檔的函數（當然該函數的原型要按照void (\*ngx_pool_cleanup_pt)(void \*data)）。


.. code-block:: none 

    void ngx_destroy_pool(ngx_pool_t *pool);

該函數就是釋放pool中持有的所有記憶體，以及依次調用cleanup欄位所管理的鏈表中每個元素的handler欄位所指向的函數，來釋放掉所有該pool管理的資源。並且把pool指向的ngx_pool_t也釋放掉了，完全不可用了。 


.. code-block:: none 

    void ngx_reset_pool(ngx_pool_t *pool);

該函數釋放pool中所有大塊記憶體鏈表上的記憶體，小塊記憶體鏈上的記憶體塊都修改為可用。但是不會去處理cleanup鏈表上的專案。 


ngx_array_t(100%)
~~~~~~~~~~~~~~~~~~~~

ngx_array_t是nginx內部使用的陣列結構。nginx的陣列結構在存儲上與大家認知的C語言內置的陣列有相似性，比如實際上存儲資料的區域也是一大塊連續的記憶體。但是陣列除了存儲資料的記憶體以外還包含一些元資訊來描述相關的一些資訊。下面我們從陣列的定義上來詳細的瞭解一下。ngx_array_t的定義位於src/core/ngx_array.c|h裏面。 

.. code-block:: none

    typedef struct ngx_array_s       ngx_array_t;
    struct ngx_array_s {
        void        *elts;
        ngx_uint_t   nelts;
        size_t       size;
        ngx_uint_t   nalloc;
        ngx_pool_t  *pool;
    };


:elts: 指向實際的資料存儲區域。 

:nelts: 陣列實際元素個數。
 
:size: 陣列單個元素的大小，單位是位元組。 

:nalloc: 陣列的容量。表示該陣列在不引發擴容的前提下，可以最多存儲的元素的個數。當nelts增長到達nalloc 時，如果再往此陣列中存儲元素，則會引發陣列的擴容。陣列的容量將會擴展到原有容量的2倍大小。實際上是分配新的一塊記憶體，新的一塊記憶體的大小是原有記憶體大小的2倍。原有的資料會被拷貝到新的一塊記憶體中。 

:pool: 該陣列用來分配記憶體的記憶體池。




下面介紹ngx_array_t相關操作函數。

.. code-block:: none

    ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);

創建一個新的陣列物件，並返回這個物件。 

:p: 陣列分配記憶體使用的記憶體池；
:n: 陣列的初始容量大小，即可以在不擴容的情況下最多可以容納的元素個數。
:size: 單個元素的大小，單位是位元組。


.. code-block:: none 

    void ngx_array_destroy(ngx_array_t *a);

銷毀該陣列物件，並釋放其對應的記憶體給對應的記憶體池。需要注意的是，調用該函數以後，陣列物件上個欄位的值並沒有被清零。所以即便這個時候物件a上各欄位還有有意義的值，但是這個物件絕對不應該被再使用了，除非是使用ngx_array_init函數。 


.. code-block:: none 

    void *ngx_array_push(ngx_array_t *a);

在陣列a上新追加一個元素，並返回指向新元素的指標。需要把返回的指標使用類型轉換，轉換為具體的類型，然後再給新元素本身或者是各欄位（如果陣列的元素是複雜類型）賦值。 


.. code-block:: none 

    void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);

在陣列a上追加n個元素，並返回指向這些追加元素的首個元素的位置的指標。 


.. code-block:: none

    static ngx_inline ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size);

如果一個陣列物件是被分配在堆上的，那麼當調用ngx_array_destroy銷毀以後，如果想再次使用，就可以調用此函數。

如果一個陣列物件是被分配在棧上的，那麼就需要調用此函數，進行初始化的工作以後，才可以使用。  

**注意事項\:** 
陣列在擴容時，舊的記憶體不會被釋放，會造成記憶體的浪費。因此，最好能提前規劃好陣列的容量，在創建或者初始化的時候一次搞定，避免多次擴容，造成記憶體浪費。



ngx_hash_t(100%)
~~~~~~~~~~~~~~~~~~

ngx_hash_t是nginx自己的hash表的實現。定義和實現位於src/core/ngx_hash.h|c中。ngx_hash_t的實現也與資料結構教課書上所描述的hash表的實現是大同小異。對於常用的解決衝突的方法有線性探測，二次探測和開鏈法等。ngx_hash_t使用的是最常用的一種，也就是開鏈法，這也是STL中的hash表使用的方法。 

但是ngx_hash_t的實現又有其幾個顯著的特點:

1. ngx_hash_t不像其他的hash表的實現，可以插入刪除元素，它只能一次初始化，就構建起整個hash表以後，既不能再刪除，也不能在插入元素了。
2. ngx_hash_t的開鏈並不是真的開了一個鏈表，實際上是開了一段連續的存儲空間，幾乎可以看做是一個陣列。這是因為ngx_hash_t在初始化的時候，會經歷一次預計算的過程，提前把每個桶裏面會有多少元素放進去給計算出來，這樣就提前知道每個桶的大小了。那麼就不需要使用鏈表，一段連續的存儲空間就足夠了。這也從一定程度上節省了記憶體的使用。

從上面的描述，我們可以看出來，實際上ngx_hash_t的使用是非常簡單。就兩步，首先是初始化，然後就可以在裏面進行查找了。下面我們詳細來看一下。

ngx_hash_t的初始化。


.. code-block:: none

    ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,  ngx_uint_t nelts);

首先我們來看一下初始化函數。該函數的第一個參數hinit是初始化的一些參數的一個集合。 names是初始化一個ngx_hash_t所需要的所有key的一個陣列。而nelts就是key的個數。下面先看一下ngx_hash_init_t類型，該類型提供了初始化一個hash表所需要的一些基本資訊。 

.. code-block:: none

    typedef struct {
        ngx_hash_t       *hash;
        ngx_hash_key_pt   key;
    
        ngx_uint_t        max_size;
        ngx_uint_t        bucket_size;
    
        char             *name;
        ngx_pool_t       *pool;
        ngx_pool_t       *temp_pool;
    } ngx_hash_init_t;


:hash: 該欄位如果為NULL，那麼調用完初始化韓式有，該欄位指向新創建出來的hash表。如果該欄位不為NULL，那麼在初始的時候，所有的資料被插入了這個欄位所指的hash表中。

:key: 指向從字串生成hash值的hash函數。nginx的源代碼中提供了默認的實現函數ngx_hash_key_lc。

:max_size: hash表中的桶的個數。該欄位越大，元素存儲時衝突的可能性越小，每個桶中存儲的元素會更少，則查詢起來的速度更快。當然，這個值越大，越造成記憶體的浪費，(實際上也浪費不了多少)。

:bucket_size: 每個桶的最大限制大小，單位是位元組。如果在初始化一個hash表的時候，發現某個桶裏面無法存的下所有屬於該桶的元素，則hash表初始化失敗。

:name: 該hash表的名字。

:pool: 該hash表分配記憶體使用的pool。  

:temp_pool: 該hash表使用的零時pool，在初始化完成以後，該pool可以被釋放和銷毀掉。


下面來看一下存儲hash表key的陣列的結構。

.. code-block:: none

    typedef struct {
        ngx_str_t         key;
        ngx_uint_t        key_hash;
        void             *value;
    } ngx_hash_key_t;


key和value的含義顯而易見，就不用解釋了。key_hash是對key使用hash函數計算出來的值。 對這兩個結構分析完成以後，我想大家應該都已經明白這個函數應該是如何使用了吧。該函數成功初始化一個hash表以後，返回NGX_OK，否則返回NGX_ERROR。



.. code-block:: none

    void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len);

在hash裏面查找key對應的value。實際上這裏的key是對真正的key（也就是name）計算出的hash值。len是name的長度。

如果查找成功，則返回指向value的指標，否則返回NULL。


ngx_hash_wildcard_t(100%)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~


nginx為了處理帶有通配符的功能變數名稱的匹配問題，實現了ngx_hash_wildcard_t這樣的hash表。他可以支援兩種類型的帶有通配符的功能變數名稱。一種是通配符在前的，例如：“\*.abc.com”，也可以省略掉星號，直接寫成”.abc.com”。這樣的key，可以匹配www.abc.com，qqq.www.abc.com之類的。另外一種是通配符在末尾的，例如：“mail.xxx.\*”，請特別注意通配符在末尾的不像位於開始的通配符可以被省略掉。這樣的通配符，可以匹配mail.xxx.com、mail.xxx.com.cn、mail.xxx.net之類的功能變數名稱。  **注意，一個ngx_hash_wildcard_t類型的hash表只能包含通配符在前的key或者是通配符在後的key。不能同時包含兩種類型的通配符的key。**


另外有一點必須說明，就是一個ngx_hash_wildcard_t類型的hash表只能包含通配符在前的key或者是通配符在後的key。不能同時包含兩種類型的通配符的key。ngx_hash_wildcard_t類型變數的構建是通過函數ngx_hash_wildcard_init完成的，而查詢是通過函數ngx_hash_find_wc_head或者ngx_hash_find_wc_tail來做的。ngx_hash_find_wc_head是查詢包含通配符在前的key的hash表的，而ngx_hash_find_wc_tail是查詢包含通配符在後的key的hash表的。

下面詳細說明這幾個函數的用法。

.. code-block:: none

    ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
        ngx_uint_t nelts);

該函數迎來構建一個可以包含通配符key的hash表。

:hint: 構造一個通配符hash表的一些參數的一個集合。關於該參數對應的類型的說明，請參見ngx_hash_t類型中ngx_hash_init函數的說明。

:names: 構造此hash表的所有的通配符key的陣列。特別要注意的是這裏的key已經都是被預處理過的。例如：“\*.abc.com”或者“.abc.com”被預處理完成以後，變成了“com.abc.”。而“mail.xxx.\*”則被預處理為“mail.xxx.”。為什麼會被處理這樣？這裏不得不簡單地描述一下通配符hash表的實現原理。當構造此類型的hash表的時候，實際上是構造了一個hash表的一個“鏈表”，是通過hash表中的key“鏈結”起來的。比如：對於“\*.abc.com”將會構造出2個hash表，第一個hash表中有一個key為com的表項，該表項的value包含有指向第二個hash表的指標，而第二個hash表中有一個表項abc，該表項的value包含有指向\*.abc.com對應的value的指針。那麼查詢的時候，比如查詢www.abc.com的時候，先查com，通過查com可以找到第二級的hash表，在第二級hash表中，再查找abc，依次類推，直到在某一級的hash表中查到的表項對應的value對應一個真正的值而非一個指向下一級hash表的指標的時候，查詢過程結束。**這裏有一點需要特別注意的，就是names陣列中元素的value所對應的值（也就是真正的value所在的地址）必須是能被4整除的，或者說是在4的倍數的地址上是對齊的。因為這個value的值的低兩位bit是有用的，所以必須為0。如果不滿足這個條件，這個hash表查詢不出正確結果。**


:nelts: names陣列元素的個數。
 

該函數執行成功返回NGX_OK，否則NGX_ERROR。




.. code-block:: none

    void *ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);



該函數查詢包含通配符在前的key的hash表的。

:hwc: hash表對象的指標。
:name: 需要查詢的功能變數名稱，例如: www.abc.com。
:len: name的長度。

該函數返回匹配的通配符對應value。如果沒有查到，返回NULL。


.. code-block:: none
    
    void *ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);

該函數查詢包含通配符在末尾的key的hash表的。
參數及返回值請參加上個函數的說明。


ngx_hash_combined_t(100%)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

組合類型hash表，該hash表的定義如下：  
.. code-block:: none

    typedef struct {
        ngx_hash_t            hash;
        ngx_hash_wildcard_t  *wc_head;
        ngx_hash_wildcard_t  *wc_tail;
    } ngx_hash_combined_t;


從其定義顯見，該類型實際上包含了三個hash表，一個普通hash表，一個包含前向通配符的hash表和一個包含後向通配符的hash表。

nginx提供該類型的作用，在於提供一個方便的容器包含三個類型的hash表，當有包含通配符的和不包含通配符的一組key構建hash表以後，以一種方便的方式來查詢，你不需要再考慮一個key到底是應該到哪個類型的hash表裏去查了。

構造這樣一組合hash表的時候，首先定義一個該類型的變數，在分別構造其包含的三個子hash表即可。

對於該類型hash表的查詢，nginx提供了一個方便的函數ngx_hash_find_combined。

.. code-block:: none

    void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key,
    u_char *name, size_t len);

該函數在此組合hash表中，依次查詢其三個子hash表，看是否匹配，一旦找到，立即返回查找結果，也就是說如果有多個可能匹配，則只返回第一個匹配的結果。

:hash: 此組合hash表物件。
:key: 根據name計算出的hash值。
:name: key的具體內容。
:len: name的長度。

返回查詢的結果，未查到則返回NULL。


ngx_hash_keys_arrays_t(100%) 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

大家看到在構建一個ngx_hash_wildcard_t的時候，需要對通配符的哪些key進行預處理。這個處理起來比較麻煩。而當有一組key，這些裏面既有無通配符的key，也有包含通配符的key的時候。我們就需要構建三個hash表，一個包含普通的key的hash表，一個包含前向通配符的hash表，一個包含後向通配符的hash表（或者也可以把這三個hash表組合成一個ngx_hash_combined_t）。在這種情況下，為了讓大家方便的構造這些hash表，nginx提供給了此輔助類型。

該類型以及相關的操作函數也定義在src/core/ngx_hash.h|c裏。我們先來看一下該類型的定義。


.. code-block:: none

    typedef struct {
        ngx_uint_t        hsize;
    
        ngx_pool_t       *pool;
        ngx_pool_t       *temp_pool;
    
        ngx_array_t       keys;
        ngx_array_t      *keys_hash;
    
        ngx_array_t       dns_wc_head;
        ngx_array_t      *dns_wc_head_hash;
    
        ngx_array_t       dns_wc_tail;
        ngx_array_t      *dns_wc_tail_hash;
    } ngx_hash_keys_arrays_t;


:hsize: 將要構建的hash表的桶的個數。對於使用這個結構中包含的資訊構建的三種類型的hash表都會使用此參數。

:pool: 構建這些hash表使用的pool。

:temp_pool: 在構建這個類型以及最終的三個hash表過程中可能用到臨時pool。該temp_pool可以在構建完成以後，被銷毀掉。這裏只是存放臨時的一些記憶體消耗。

:keys: 存放所有非通配符key的陣列。

:keys_hash: 這是個二維陣列，第一個維度代表的是bucket的編號，那麼keys_hash[i]中存放的是所有的key算出來的hash值對hsize取模以後的值為i的key。假設有3個key,分別是key1,key2和key3假設hash值算出來以後對hsize取模的值都是i，那麼這三個key的值就順序存放在keys_hash[i][0],keys_hash[i][1], keys_hash[i][2]。該值在調用的過程中用來保存和檢測是否有衝突的key值，也就是是否有重複。

:dns_wc_head: 放前向通配符key被處理完成以後的值。比如：“\*.abc.com” 被處理完成以後，變成 “com.abc.” 被存放在此陣列中。

:dns_wc_tail: 存放後向通配符key被處理完成以後的值。比如：“mail.xxx.\*” 被處理完成以後，變成 “mail.xxx.” 被存放在此陣列中。

:dns_wc_head_hash: 該值在調用的過程中用來保存和檢測是否有衝突的前向通配符的key值，也就是是否有重複。

:dns_wc_tail_hash: 該值在調用的過程中用來保存和檢測是否有衝突的後向通配符的key值，也就是是否有重複。




在定義一個這個類型的變數，並對欄位pool和temp_pool賦值以後，就可以調用函數ngx_hash_add_key把所有的key加入到這個結構中了，該函數會自動實現普通key，帶前向通配符的key和帶後向通配符的key的分類和檢查，並將這個些值存放到對應的欄位中去，
然後就可以通過檢查這個結構體中的keys、dns_wc_head、dns_wc_tail三個陣列是否為空，來決定是否構建普通hash表，前向通配符hash表和後向通配符hash表了（在構建這三個類型的hash表的時候，可以分別使用keys、dns_wc_head、dns_wc_tail三個陣列）。

構建出這三個hash表以後，可以組合在一個ngx_hash_combined_t物件中，使用ngx_hash_find_combined進行查找。或者是仍然保持三個獨立的變數對應這三個hash表，自I機決定何時以及在哪個hash表中進行查詢。

.. code-block:: none

    ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);  

初始化這個結構，主要是對這個結構中的ngx_array_t類型的欄位進行初始化，成功返回NGX_OK。

:ha: 該結構的物件指標。

:type: 該欄位有2個值可選擇，即NGX_HASH_SMALL和NGX_HASH_LARGE。用來指明將要建立的hash表的類型，如果是NGX_HASH_SMALL，則有比較小的桶的個數和陣列元素大小。NGX_HASH_LARGE則相反。

.. code-block:: none

    ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key,
    void *value, ngx_uint_t flags);

一般是迴圈調用這個函數，把一組鍵值對加入到這個結構體中。返回NGX_OK是加入成功。返回NGX_BUSY意味著key值重複。

:ha: 該結構的物件指標。

:key: 參數名自解釋了。

:value: 參數名自解釋了。

:flags: 有兩個標誌位元可以設置，NGX_HASH_WILDCARD_KEY和NGX_HASH_READONLY_KEY。同時要設置的使用邏輯與操作符就可以了。NGX_HASH_READONLY_KEY被設置的時候，在計算hash值的時候，key的值不會被轉成小寫字元，否則會。NGX_HASH_WILDCARD_KEY被設置的時候，說明key裏面可能含有通配符，會進行相應的處理。如果兩個標誌位元都不設置，傳0。


有關於這個資料結構的使用，可以參考src/http/ngx_http.c中的ngx_http_server_names函數。


ngx_chain_t(100%) 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



nginx的filter模組在處理從別的filter模組或者是handler模組傳遞過來的資料（實際上就是需要發送給用戶端的http response）。這個傳遞過來的資料是以一個鏈表的形式(ngx_chain_t)。而且資料可能被分多次傳遞過來。也就是多次調用filter的處理函數，以不同的ngx_chain_t。

該結構被定義在src/core/ngx_buf.h|c。下面我們來看一下ngx_chain_t的定義。

.. code-block:: none

    struct ngx_chain_s {
        ngx_buf_t    *buf;
        ngx_chain_t  *next;
    };


就2個欄位，next指向這個鏈表的下個節點。buf指向實際的資料。所以在這個鏈表上追加節點也是非常容易，只要把末尾元素的next指標指向新的節點，把新節點的next賦值為NULL即可。

.. code-block:: none

    ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool);

該函數創建一個ngx_chain_t的物件，並返回指向物件的指標，失敗返回NULL。

.. code-block:: none

    #define ngx_free_chain(pool, cl)                                             \
        cl->next = pool->chain;                                                  \
    pool->chain = cl

該巨集釋放一個ngx_chain_t類型的物件。如果要釋放整個chain，則迭代此鏈表，對每個節點使用此宏即可。

**注意\:** 對ngx_chaint_t類型的釋放，並不是真的釋放了記憶體，而僅僅是把這個物件掛在了這個pool物件的一個叫做chain的欄位對應的chain上，以供下次從這個pool上分配ngx_chain_t類型物件的時候，快速的從這個pool->chain上取下鏈首元素就返回了，當然，如果這個鏈是空的，才會真的在這個pool上使用ngx_palloc函數進行分配。 




ngx_buf_t(99%) 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



這個ngx_buf_t就是這個ngx_chain_t鏈表的每個節點的實際資料。該結構實際上是一種抽象的資料結構，它代表某種具體的資料。這個資料可能是指向記憶體中的某個緩衝區，也可能指向一個檔的某一部分，也可能是一些純元資料（元資料的作用在於指示這個鏈表的讀取者對讀取的資料進行不同的處理）。

該資料結構位於src/core/ngx_buf.h|c文件中。我們來看一下它的定義。

.. code-block:: none

    struct ngx_buf_s {
        u_char          *pos;
        u_char          *last;
        off_t            file_pos;
        off_t            file_last;
    
        u_char          *start;         /* start of buffer */
        u_char          *end;           /* end of buffer */
        ngx_buf_tag_t    tag;
        ngx_file_t      *file;
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
    
        unsigned         recycled:1;
        unsigned         in_file:1;
        unsigned         flush:1;
        unsigned         sync:1;
        unsigned         last_buf:1;
        unsigned         last_in_chain:1;
    
        unsigned         last_shadow:1;
        unsigned         temp_file:1;
    
        /* STUB */ int   num;
    };

:pos: 當buf所指向的資料在記憶體裏的時候，pos指向的是這段資料開始的位置。

:last: 當buf所指向的資料在記憶體裏的時候，last指向的是這段資料結束的位置。

:file_pos: 當buf所指向的資料是在檔裏的時候，file_pos指向的是這段資料的開始位置在檔中的偏移量。

:file_last: 當buf所指向的資料是在檔裏的時候，file_last指向的是這段資料的結束位置在檔中的偏移量。

:start: 當buf所指向的資料在記憶體裏的時候，這一整塊記憶體包含的內容可能被包含在多個buf中(比如在某段資料中間插入了其他的資料，這一塊資料就需要被拆分開)。那麼這些buf中的start和end都指向這一塊記憶體的開始位址和結束位址。而pos和last指向本buf所實際包含的資料的開始和結尾。

:end: 解釋參見start。

:tag: 實際上是一個void\*類型的指標，使用者可以關聯任意的物件上去，只要對使用者有意義。

:file: 當buf所包含的內容在檔中是，file欄位指向對應的檔物件。

:shadow: 當這個buf完整copy了另外一個buf的所有欄位的時候，那麼這兩個buf指向的實際上是同一塊記憶體，或者是同一個檔的同一部分，此時這兩個buf的shadow欄位都是指向對方的。那麼對於這樣的兩個buf，在釋放的時候，就需要使用者特別小心，具體是由哪里釋放，要提前考慮好，如果造成資源的多次釋放，可能會造成程式崩潰！

:temporary: 為1時表示該buf所包含的內容是在一個用戶創建的記憶體塊中，並且可以被在filter處理的過程中進行變更，而不會造成問題。

:memory: 為1時表示該buf所包含的內容是在記憶體中，但是這些內容確不能被進行處理的filter進行變更。

:mmap: 為1時表示該buf所包含的內容是在記憶體中, 是通過mmap使用記憶體映射從檔中映射到記憶體中的，這些內容確不能被進行處理的filter進行變更。

:recycled: 可以回收的。也就是這個buf是可以被釋放的。這個欄位通常是配合shadow欄位一起使用的，對於使用ngx_create_temp_buf 函數創建的buf，並且是另外一個buf的shadow，那麼可以使用這個欄位來標示這個buf是可以被釋放的。

:in_file: 為1時表示該buf所包含的內容是在檔中。

:flush: 遇到有flush欄位被設置為1的的buf的chain，則該chain的資料即便不是最後結束的資料（last_buf被設置，標誌所有要輸出的內容都完了），也會進行輸出，不會受postpone_output配置的限制，但是會受到發送速率等其他條件的限制。

:sync:

:last_buf: 資料被以多個chain傳遞給了篩檢程式，此欄位為1表明這是最後一個buf。

:last_in_chain: 在當前的chain裏面，此buf是最後一個。特別要注意的是last_in_chain的buf不一定是last_buf，但是last_buf的buf一定是last_in_chain的。這是因為資料會被以多個chain傳遞給某個filter模組。

:last_shadow:  在創建一個buf的shadow的時候，通常將新創建的一個buf的last_shadow置為1。 


:temp_file:  由於受到記憶體使用的限制，有時候一些buf的內容需要被寫到磁片上的暫存檔案中去，那麼這時，就設置此標誌
 。


對於此物件的創建，可以直接在某個ngx_pool_t上分配，然後根據需要，給對應的欄位賦值。也可以使用定義好的2個宏：

.. code-block:: none

    #define ngx_alloc_buf(pool)  ngx_palloc(pool, sizeof(ngx_buf_t))
    #define ngx_calloc_buf(pool) ngx_pcalloc(pool, sizeof(ngx_buf_t))


這兩個巨集使用類似函數，也是不說自明的。

對於創建temporary欄位為1的buf（就是其內容可以被後續的filter模組進行修改），可以直接使用函數ngx_create_temp_buf進行創建。

.. code-block:: none

    ngx_buf_t *ngx_create_temp_buf(ngx_pool_t *pool, size_t size);


該函數創建一個ngx_but_t類型的物件，並返回指向這個物件的指標，創建失敗返回NULL。

對於創建的這個物件，它的start和end指向新分配記憶體開始和結束的地方。pos和last都指向這塊新分配記憶體的開始處，這樣，後續的操作可以在這塊新分配的記憶體上存入資料。

:pool: 分配該buf和buf使用的記憶體所使用的pool。
:size: 該buf使用的記憶體的大小。



為了配合對ngx_buf_t的使用，nginx定義了以下的宏方便操作。

.. code-block:: none

    #define ngx_buf_in_memory(b)        (b->temporary || b->memory || b->mmap)

返回這個buf裏面的內容是否在記憶體裏。

.. code-block:: none

    #define ngx_buf_in_memory_only(b)   (ngx_buf_in_memory(b) && !b->in_file)

返回這個buf裏面的內容是否僅僅在記憶體裏，並且沒有在檔裏。

.. code-block:: none

    #define ngx_buf_special(b)                                                   \
        ((b->flush || b->last_buf || b->sync)                                    \
         && !ngx_buf_in_memory(b) && !b->in_file)

返回該buf是否是一個特殊的buf，只含有特殊的標誌和沒有包含真正的資料。

.. code-block:: none

    #define ngx_buf_sync_only(b)                                                 \
        (b->sync                                                                 \
         && !ngx_buf_in_memory(b) && !b->in_file && !b->flush && !b->last_buf)

返回該buf是否是一個隻包含sync標誌而不包含真正資料的特殊buf。

.. code-block:: none

    #define ngx_buf_size(b)                                                      \
        (ngx_buf_in_memory(b) ? (off_t) (b->last - b->pos):                      \
                                (b->file_last - b->file_pos))


返回該buf所含資料的大小，不管這個資料是在檔裏還是在記憶體裏。





ngx_list_t(100%) 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


ngx_list_t顧名思義，看起來好像是一個list的資料結構。這樣的說法，算對也不算對。因為它符合list類型資料結構的一些特點，比如可以添加元素，實現自增長，不會像陣列類型的資料結構，受到初始設定的陣列容量的限制，並且它跟我們常見的list型資料結構也是一樣的，內部實現使用了一個鏈表。

那麼它跟我們常見的鏈表實現的list有什麼不同呢？不同點就在於它的節點，它的節點不像我們常見的list的節點，只能存放一個元素，ngx_list_t的節點實際上是一個固定大小的陣列。

在初始化的時候，我們需要設定元素需要佔用的空間大小，每個節點陣列的容量大小。在添加元素到這個list裏面的時候，會在最尾部的節點裏的陣列上添加元素，如果這個節點的陣列存滿了，就再增加一個新的節點到這個list裏面去。

好了，看到這裏，大家應該基本上明白這個list結構了吧？還不明白也沒有關係，下面我們來具體看一下它的定義，這些定義和相關的操作函數定義在src/core/ngx_list.h|c文件中。

.. code-block:: none

    typedef struct {
        ngx_list_part_t  *last;
        ngx_list_part_t   part;
        size_t            size;
        ngx_uint_t        nalloc;
        ngx_pool_t       *pool;
    } ngx_list_t;

:last: 指向該鏈表的最後一個節點。
:part: 該鏈表的首個存放具體元素的節點。
:size: 鏈表中存放的具體元素所需記憶體大小。
:nalloc: 每個節點所含的固定大小的陣列的容量。
:pool: 該list使用的分配記憶體的pool。

好，我們在看一下每個節點的定義。

.. code-block:: none

    typedef struct ngx_list_part_s  ngx_list_part_t;
    struct ngx_list_part_s {
        void             *elts;
        ngx_uint_t        nelts;
        ngx_list_part_t  *next;
    };


:elts: 節點中存放具體元素的記憶體的開始位址。

:nelts: 節點中已有元素個數。這個值是不能大於鏈表頭節點ngx_list_t類型中的nalloc欄位的。

:next: 指向下一個節點。


我們來看一下提供的一個操作的函數。

.. code-block:: none

    ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);

該函數創建一個ngx_list_t類型的物件,並對該list的第一個節點分配存放元素的記憶體空間。

:pool: 分配記憶體使用的pool。

:n: 每個節點固定長度的陣列的長度。

:size: 存放的具體元素的個數。

:返回值: 成功返回指向創建的ngx_list_t對象的指標，失敗返回NULL。

.. code-block:: none

    void *ngx_list_push(ngx_list_t *list);

該函數在給定的list的尾部追加一個元素，並返回指向新元素存放空間的指標。如果追加失敗，則返回NULL。

.. code-block:: none

    static ngx_inline ngx_int_t
    ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size);

該函數是用於ngx_list_t類型的物件已經存在，但是其第一個節點存放元素的記憶體空間還未分配的情況下，可以調用此函數來給這個list的首節點來分配存放元素的記憶體空間。

那麼什麼時候會出現已經有了ngx_list_t類型的物件，而其首節點存放元素的記憶體尚未分配的情況呢？那就是這個ngx_list_t類型的變數並不是通過調用ngx_list_create函數創建的。例如：如果某個結構體的一個成員變數是ngx_list_t類型的，那麼當這個結構體類型的物件被創建出來的時候，這個成員變數也被創建出來了，但是它的首節點的存放元素的記憶體並未被分配。

總之，如果這個ngx_list_t類型的變數，如果不是你通過調用函數ngx_list_create創建的，那麼就必須調用此函數去初始話，否則，你往這個list裏追加元素就可能引發不可預知的行為，亦或程式會崩潰!



nginx的配置系統
------------------------



指令概述
~~~~~~~~~~~~~~~~~~~~



指令參數
~~~~~~~~~~~~~~~~~~~~



指令上下文
~~~~~~~~~~~~~~~~~~~~~~~



nginx的請求處理
------------------------



請求的處理流程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



nginx的模組化體系結構
---------------------------------



模組概述
------------------



模組的分類
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



