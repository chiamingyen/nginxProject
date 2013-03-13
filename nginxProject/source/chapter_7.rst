模組開發高級篇(30%)
===============================


變數(80%)
----------------


綜述
+++++++++++++++++++++++++++
在Nginx中同一個請求需要在模組之間資料的傳遞或者說在配置檔裏面使用模組動態的資料一般來說都是使用變數，比如在HTTP模組中導出了host/remote_addr等變數，這樣我們就可以在配置檔中以及在其他的模組使用這個變數。在Nginx中，有兩種定義變數的方式，一種是在配置檔中,使用set指令，一種就是上面我們提到的在模組中定義變數，然後導出.

在Nginx中所有的變數都是與HTTP相關的(也就是說賦值都是在請求階段)，並且基本上是同時保存在兩個資料結構中，一個就是hash表(可選)，另一個是陣列. 比如一些特殊的變數，比如arg_xxx/cookie_xxx等，這些變數的名字是不確定的(因此不能內置)，而且他們還是唯讀的(不能交由用戶修改)，如果每個都要放到hash表中的話(不知道用戶會取多少個),會很占空間的，因此這些變數就沒有hash,只有索引.這裏要注意，用戶不能定義這樣的變數，這樣的變數只存在於Nginx內部.

對應的變數結構體是這樣子(每一個變數都是一個ngx_http_variable_s結構體)的：

.. code-block:: none

                struct ngx_http_variable_s {
                    ngx_str_t                     name;   /* must be first to build the hash */
                    ngx_http_set_variable_pt      set_handler;
                    ngx_http_get_variable_pt      get_handler;
                    uintptr_t                     data;
                    ngx_uint_t                    flags;
                    ngx_uint_t                    index;
                };

其中name表示對應的變數名字，set/get_handler表示對應的設置以及讀取回調，而data是傳遞給回調的參數，flags表示變數的屬性，index提供了一個索引(陣列的腳標)，從而可以迅速定位到對應的變數。set/get_handler只有在真正讀取設置變數的時候才會被調用.

這裏要注意flag屬性,flag屬性就是由下面的幾個屬性組合而成:

.. code-block:: none

                #define NGX_HTTP_VAR_CHANGEABLE   1
                #define NGX_HTTP_VAR_NOCACHEABLE  2
                #define NGX_HTTP_VAR_INDEXED      4
                #define NGX_HTTP_VAR_NOHASH       8

1. NGX_HTTP_VAR_CHANGEABLE表示這個變數是可變的,比如arg_xxx這類變數，如果你使用set指令來修改，那麼Nginx就會報錯.
2. NGX_HTTP_VAR_NOCACHEABLE表示這個變數每次都要去取值，而不是直接返回上次cache的值(配合對應的介面).
3. NGX_HTTP_VAR_INDEXED表示這個變數是用索引讀取的.
4. NGX_HTTP_VAR_NOHASH表示這個變數不需要被hash.

而變數在Nginx中的初始化流程是這樣的:

1. 首先當解析HTTP之前會調用ngx_http_variables_add_core_vars(pre_config)來將HTTP core模組導出的變數(http_host/remote_addr...)添加進全局的hash key鏈中.

2. 解析完HTTP模組之後，會調用ngx_http_variables_init_vars來初始化所有的變數(不僅包括HTTP core模組的變數，也包括其他的HTTP模組導出的變數,以及配置檔中使用set命令設置的變數),這裏的初始化包括初始化hash表,以及初始化陣列索引.

3. 當每次請求到來時會給每個請求創建一個變數陣列(陣列的個數就是上面第二步所保存的變數個數)。然後只有取變數值的時候，才會將變數保存在對應的變數陣列位置。

創建變數
+++++++++++++++++++++++++++
在Nginx中，創建變數有兩種方式，分別是在配置檔中使用set指令，和在模組中調用對應的介面，在配置檔中創建變數比較簡單，因此我們主要來看如何在模組中創建自己的變數。

在Nginx中提供了下面的介面，可以供模組調用來創建變數。

.. code-block:: none

                ngx_http_variable_t *ngx_http_add_variable(ngx_conf_t *cf, ngx_str_t *name, ngx_uint_t flags);

這個函數所做的工作就是將變數 "name"添加進全局的hash key表中,然後初始化一些域，不過這裏要注意，對應的變數的get/set回調，需要當這個函數返回之後，顯示的設置,比如在split_clients模組中的例子:

.. code-block:: none

                var = ngx_http_add_variable(cf, &name, NGX_HTTP_VAR_CHANGEABLE);
                if (var == NULL) {
                        return NGX_CONF_ERROR;
                }
                //設置回調
                var->get_handler = ngx_http_split_clients_variable;
                var->data = (uintptr_t) ctx;

而對應的回調函數原型是這樣的:

.. code-block:: none

                typedef void (*ngx_http_set_variable_pt) (ngx_http_request_t *r,
                    ngx_http_variable_value_t *v, uintptr_t data);
                typedef ngx_int_t (*ngx_http_get_variable_pt) (ngx_http_request_t *r,
                    ngx_http_variable_value_t *v, uintptr_t data);

回調函數比較簡單，第一個參數是當前請求，第二個是需要設置或者獲取的變數值，第三個是初始化時的回調指標，這裏我們著重來看一下ngx_http_variable_value_t,下面就是這個結構體的原型:

.. code-block:: none

                typedef struct {
                    unsigned    len:28;

                    unsigned    valid:1;
                    unsigned    no_cacheable:1;
                    unsigned    not_found:1;
                    unsigned    escape:1;
                    u_char     *data;
                } ngx_variable_value_t;

這裏主要是data域，當我們在get_handle中設置變數值的時候，只需要將對應的值放入到data中就可以了，這裏data需要在get_handle中分配記憶體,比如下面的例子(ngx_http_fastcgi_script_name_variable),就是fastcgi_script_name變數的get_handler代碼片段:

.. code-block:: none

                v->len = f->script_name.len + flcf->index.len;

                v->data = ngx_pnalloc(r->pool, v->len);
                if (v->data == NULL) {
                        return NGX_ERROR;
                }

                p = ngx_copy(v->data, f->script_name.data, f->script_name.len);
                ngx_memcpy(p, flcf->index.data, flcf->index.len);


使用變數
+++++++++++++++++++++++++++

Nginx的內部變數指的就是Nginx的官方模組中所導出的變數，在Nginx中，大部分常用的變數都是CORE HTTP模組導出的。而在Nginx中，不僅可以在模組代碼中使用變數，而且還可以在配置檔中使用。

假設我們需要在配置檔中使用http模組的host變數，那麼只需要這樣在變數名前加一個$符號就可以了($host).而如果需要在模組中使用host變數，那麼就比較麻煩，Nginx提供了下面幾個介面來取得變數:

.. code-block:: none

                ngx_http_variable_value_t *ngx_http_get_indexed_variable(ngx_http_request_t *r,
                    ngx_uint_t index);
                ngx_http_variable_value_t *ngx_http_get_flushed_variable(ngx_http_request_t *r,
                    ngx_uint_t index);
                ngx_http_variable_value_t *ngx_http_get_variable(ngx_http_request_t *r,
                    ngx_str_t *name, ngx_uint_t key);

他們的區別是這樣子的，ngx_http_get_indexed_variable和ngx_http_get_flushed_variable都是用來取得有索引的變數，不過他們的區別是後一個會處理
NGX_HTTP_VAR_NOCACHEABLE這個標記，也就是說如果你想要cache你的變數值，那麼你的變數屬性就不能設置NGX_HTTP_VAR_NOCACHEABLE,並且通過ngx_http_get_flushed_variable來獲取變數值.而ngx_http_get_variable和上面的區別就是它能夠得到沒有索引的變數值.

通過上面我們知道可以通過索引來得到變數值，可是這個索引改如何取得呢，Nginx也提供了對應的介面：

.. code-block:: none

                ngx_int_t ngx_http_get_variable_index(ngx_conf_t *cf, ngx_str_t *name);


通過這個介面，就可以取得對應變數名的索引值。

接下來來看對應的例子，比如在http_log模組中，如果在log_format中配置了對應的變數，那麼它會調用ngx_http_get_variable_index來保存索引:

.. code-block:: none

                static ngx_int_t
                ngx_http_log_variable_compile(ngx_conf_t *cf, ngx_http_log_op_t *op,
                    ngx_str_t *value)
                {
                    ngx_int_t  index;
                    //得到變數的索引
                    index = ngx_http_get_variable_index(cf, value);
                    if (index == NGX_ERROR) {
                        return NGX_ERROR;
                    }

                    op->len = 0;
                    op->getlen = ngx_http_log_variable_getlen;
                    op->run = ngx_http_log_variable;
                    //保存索引值
                    op->data = index;

                    return NGX_OK;
                 }

然後http_log模組會使用ngx_http_get_indexed_variable來得到對應的變數值,這裏要注意，就是使用這個介面的時候，判斷返回值，不僅要判斷是否為空，也需要判斷value->not_found,這是因為只有第一次調用才會返回空，後續返回就不是空，因此需要判斷value->not_found:

.. code-block:: none

                static u_char *
                ngx_http_log_variable(ngx_http_request_t *r, u_char *buf, ngx_http_log_op_t *op)
                {
                    ngx_http_variable_value_t  *value;
                    //獲取變數值
                    value = ngx_http_get_indexed_variable(r, op->data);

                    if (value == NULL || value->not_found) {
                            *buf = '-';
                            return buf + 1;
                    }

                    if (value->escape == 0) {
                            return ngx_cpymem(buf, value->data, value->len);

                    } else {
                            return (u_char *) ngx_http_log_escape(buf, value->data, value->len);
                    }
                 }


upstream
------------------

使用subrequest訪問upstream
+++++++++++++++++++++++++++


超越upstream
+++++++++++++++++++++++++++


event機制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


例講（主動健康檢查模組）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



使用lua模組
-------------------



