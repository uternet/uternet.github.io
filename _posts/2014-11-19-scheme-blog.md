---
layout: post
title:  用 scheme 写了个玩具博客
date:   2014-11-19
---

一直觉得 web 比较神秘，前段时间看 Racket Documents 上面的这个 [tut](http://docs.racket-lang.org/continue/),并且照着写了一些代码。

Racket 里的 web server 模块提供了一些方便 web 开发的东西，比如 xexpr 表达式把 html 代码和 S-expression 统一在了一起。不过，我好象不太习惯这种方便。大多数时候，我搞不清楚我写的是 scheme 代码，还是 html 代码，需要 quote 还是不需要。写着写着就糊涂了。

我决定追寻最本质的 CGI 编程方法。找了一些 cgi 示例，几乎全是用 perl 示范的，少数还有一些用 c 写的。算是把 CGI 的运行原理基本弄清了。web server 在前台运行，回应用户的请求。如果发现用户提交的请求指向某个 CGI 程序，则运行这个程序，程序处理完成后，再通过标准输出把结果交还给 web server, 再由 web server 返回给用户的浏览器。因为后台的 CGI 程序仅仅依赖标准输入和输出来与 web server 交换数据，所以，几乎所有的编程语言都可以用来写 cgi 程序。 God, 我居然看到有人直接用 shell 脚本来写网站。

传统的 CGI 有安全和性能方面的缺陷，新的 CGI 有 FashCGI、SCGI 等，从原理上来讲，与传统的 CGI 很相似，但是架构上，增加了一层 CGI server。CGI server 作为守护进程在后台运行（也可以运行在别的机器上）前台的 web server 充当了 CGI client，负责将用户的动态请求转发给 CGI SERVER, 而 CGI server 在处理完用户请求后并不退出，而是继续运行，等待下一次调用。这样就不用不停地 fork 子进程，性能提升显著。

FashCGI似乎比较复杂，有人说这个协议太丑陋，而 SCGI 就比较简单，于是我转而寻找 SCGI 方面的资料。SCGI 最初是由 Python 实现的 CGI 协议，Apache 侧的 CGI client 是用 c 实现的。因为协议比较简单，所以，很多语言都有移植。

正好有人已经用 Racket 实现了一个 SCGI server，它位于 Planet 里，Planet 是一个社区代码库，相当于 perl 的 CPAN，别人写好的模块，只要在自己的代码里 require 一下就可以用了。

开始动手写，刚开始模仿开头提到的 tut 里的代码，实现同样的功能，可是写着写着便觉得不爽了。一是功能太简陋，二是有些代码不好翻译。比如原文里，可以简单地将一个链接指向一个函数，而用 CGI 就不行，必须判断 URI 再来决定要调用的函数。于是我决定抛开原文，自己根据功能需求来实现。很多东西不懂，边看文档边写，有些功能是写着写着又决定加上去的。所以，下面这一坨代码与其说是写成的，不如说是改成的。写到后面，已经混乱得连我自己都不愿意去看了。

先说说实现的功能及想法：

1. 用户及会话管理，用户管理是我自己加上去的。既然要管理用户，那就要涉及到 cookies 。以前一直不懂这玩意儿怎么验证用户的登陆状态。如果把用户口令保存在 cookies 中，总觉得不安全，至少要强加密吧？Racket 里没有内置可逆的强加密算法，我曾经雄心勃勃地想自己写一个 AES 加密模块，在被文档搞晕了头之后我决定用折衷的方法，就是 session。服务端判断用户的每一次点击，在后台记录并更新会话，如果用户长时间没有动作（超时时间可以设置），则会话失效，需要重新登陆。

2. 绕过一个很大的弯路：我自己实现了一个 GB2312->utf-8 的转换程序，原因是我发现 Firefox post上去的数据是用 GB2312 编码的，Racket 不能处理 GB 码。GB2312 到 UTF-8的转换需要两个步骤：首先是查表法，将 GB2312 转换成 Unicode，然后再通过算法把 Unicode 转换成 UTF-8。我查资料自己弄出来后颇有成就感，可是后来发现，可以在 html 表单里添加 accept-charset="utf-8" 来让浏览器 post UTF-8 编码的数据。所以，我白忙活一场，代码最终又拿掉了。

3. 曾经想写一个函数处理用户 post 的数据，在保留用户提交 html 代码能力的同时能提供一些方便书写的帮助及安全方面的限制，最好能处理 Markdown 的一部分语法。这几个目标有点冲突，保留 html 能力会造成一些困扰，比如用户想输入 `#include <stdio.h>` ，结果后面的 stdio.h 会显示不出来，因为浏览器会把尖括号中的内容识别成 html 标签，对于不合法的标签浏览器就不显示，于是这部分内容就被吃掉了。如果将所有的尖括号都替换成  &lt; &gt; 那么用户将失去所有的格式控制能力，连分个段落都办不到。这部分有点复杂，暂时不想实现，我干脆就不处理了，连匿名用户提交的文章评论都没有处理，如果一个恶意的用户提交危险的 html 代码也会被原样写进数据库。所以，这个 blog 只是个玩具 blog，千万不能挂到公网上去。

不管怎么说，这是我的第一个 CGI 程序，还是放在这里出出丑吧。

```scheme
#!/usr/bin/env racket
#lang racket/base

(require racket/port
         racket/list
         racket/string
         racket/date
         db
         file/md5
         (planet neil/scgi:2:0))

;;;;;;;;;;;;;;;;;;;;;;;;;;;; site-configure ;;;;;;;;;;;;;;;;;;;;;;;;;

;; 站点名字
(define site-name "site-name")

;; 初始用户名
(define site-admin "admin")

;; 显示在“作者”栏的名字
(define admin-display "admin")

;; 初始密码
(define admin-passwd "123456")

;; 数据库文件
(define db-file "blog.db")

;; 每面显示的文章数
(define pagenation 5)

;; 会话超时时间（单位：秒）
(define timeout 1800)


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;  html ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (render-page-head title dbc uid)
  (let ((head (string-append
               "<html>"
               "<head>"
               "<meta http-equiv=\"content-type\" content=\"text/html; charset=utf-8\"/>"
               "<link rel=\"stylesheet\" href=\"/css/style.css\">"
               "<title>~a</title>"
               "</head>"
               "<body>"
               "<div id=\"main\">")))
    (printf "Content-type: text/html\n\n")
    (printf head title)
    (render-banner dbc uid)))

(define (render-banner dbc uid)
  (let ((banner_ (string-append
               "<div id=\"banner\">"
               "<div id=\"site-title\"><h1>~a</h1></div>"
               "<div id=\"entrance\">~a</div>"
               )))
    (let ((user-info
           (if uid
               (string-append (uid->user dbc uid) " [<a href=\"/logout\">Logout</a>]")
               "[<a href=\"/login\">Login</a>]")))
      (printf banner_ site-name user-info))
    (render-menu)
    (printf "</div>")))

(define (render-menu)
  (let ((menu (string-append
               "<div id=\"menu\">"
               "<ul id=\"menu-items\">"
               "<li class=\"menu-item\"><a href=\"/\">主页</a></li>"
               ;;"<li class=\"menu-item\"><a href=\"/archive\">归档</a></li>"
               ;;"<li class=\"menu-item\"><a href=\"/classify\">分类</a></li>"
               "<li class=\"menu-item\"><a href=\"/search\">搜索</a></li>"
               "<li class=\"menu-item\"><a href=\"/about\">关于</a></li>"
               "</ul></div>"
               "<div class=\"clear\"></div>")))
    (printf menu)))

(define (render-page-footer)
  (let ((footer (string-append
               "<div id=\"footer\">"
               "<br><hr>"
               "Copyright(C) xxxxxx" ; 版权信息
               "</div>\n")))
    (printf footer)))

(define (add-post-form)
  (let ((form (string-append
               "<hr>"
               "<form class=\"post-from\" action=\"/add-post\" method=\"post\" accept-charset=\"utf-8\">"
               "<div><label>添加新文章</label></div>"               
               "<input class=\"short\" name=\"title\">"
               "<label for=\"title\">  标题</label>"
               "<textarea id=\"post-body\" name=\"body\"></textarea>"
               "<div class=\"clear\"></div>"
               "<div><input type=\"submit\" value=\"添加文章\"""></div>"
               "</form>")))
    (printf form)))

(define (add-comment-form pid)
  (let ((form (string-append
               "<form  class=\"post-from\" action=\"/add-comment/~a\" method=\"post\" accept-charset=\"utf-8\">"
               "<input class=\"short\" name=\"email\">"
               "<label for=\"email\">E-mail</label>"
               "<textarea name=\"content\" id=\"comment\"></textarea>"
               "<div><input type=\"submit\" value=\"添加评论\"></div>"
               "</form>")))
    (printf form (number->string pid))))

(define (page-end)
  (printf "</div></body></html>"))
  
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;
;;                             404
;;                               
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
(define (render-404)
  (printf "Status: 404 Not Found\n\n")
  (printf "HTTP 404 - Page Not Found"))
  
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;                        显示主页
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (render-posts-list dbc page)
  (let* ((id-list (reverse (all-posts dbc)))
         (len (length id-list)))
    (if (>= (* (- page 1) pagenation) len)
        (render-404)
        (begin
          (let iter ((idx (* (- page 1) pagenation)) (cnt pagenation))
            (unless (or (>= idx len) (<= cnt 0))
                    (let* ((id (list-ref id-list idx))
                           (href (string-append
                                  "/post/"
                                  (number->string id)))
                           (date (id->date dbc id))
                           (title (id->title dbc id)))
                      (printf "<li><time>~a</time> <a href=~s>~a</a></li>"
                              date href title))
                    (iter (+ idx 1) (- cnt 1))))
          (printf "<div class=\"blank\" ></div><div id=pager>[ ")
          (let iter ((cnt 1) (len len))
            (unless (<= len 0)
                    (if (= cnt page)
                        (printf " ~a " cnt)
                        (printf " <a href=~s>~a</a>"
                                   (string-append "/page/"
                                                  (number->string cnt))
                                   cnt))
                    (iter (+ cnt 1) (- len pagenation))))
          (printf " ]</div>")))))

;;显示分页
(define (render-page dbc uid . page)
  (render-page-head site-name dbc uid)
  (printf "<div class=\"postList\"><ul>")
  (if (null? page)
      (render-posts-list dbc 1)
      (render-posts-list dbc (car page)))
  (printf "</ul></div>")
  (when uid
        (add-post-form))
  (render-page-footer)
  (page-end))


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;                        显示文章页面
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (render-comment str)
  (printf "<div class=comment>~a</div>" str))

(define (render-post-comments db id)
  (printf "<div id=\"comments\">")
  (map render-comment (id->comments db id))
  (printf "</div>"))

(define (render-post-page dbc id uid)
  (let ((body (id->body dbc id))
        (title (id->title dbc id))
        (writer (uid->dname dbc (id->writer dbc id)))
        (date (id->date dbc id)))
    (render-page-head title dbc uid)
    (printf "<div id=\"post-title\"><h1>~a</h1></div>" title)
    (printf "<div id=\"post-info\">作者：~a 日期：~a</div>" writer date)
    (printf "<div id=\"post-body\">~a</div>" body)
    (printf "<div class=\"clear\"></div>")
    (printf "<hr>")
    (add-comment-form id)
    (render-post-comments dbc id)
    (render-page-footer)
    (page-end)))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; login window ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (render-login-page dbc)
  (render-page-head "Login" dbc #f)
  (printf (string-append
           "<div id=\"login\">"
           "<form action=\"/login\" method=\"post\" accept-charset=\"utf-8\">"
           "<div class=\"login-l\">User Name:</div>"               
           "<div class=\"login-r\"><input name=\"uname\"></div>"
           "<div class=\"clear\"></div>"
           "<div class=\"login-l\">Password: </div>"
           "<div class=\"login-r\"><input type=\"password\" name=\"passwd\"></div>"
           "<div class=\"clear\"></div>"
           "<div class=\"login-r\"><input type=\"submit\" value=\"Login\"></div>"
           "</form></div>"))
  (render-page-footer)
  (page-end))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; search page ;;;;;;;;;;;;;;;;;;;;;;;;;;;
(define (render-search-head dbc uid)
  (render-page-head site-name dbc uid)
  (printf (string-append
           "<div class=\"blank\"></div>"
           "<form id=searchbox action=\"search\">"
           "<input name=\"title\">"
           "<input type=\"submit\" value=\"搜索\">"
           "</form>")))

(define (single-item dbc id)
  (let ((title (id->title dbc id))
        (date (id->date dbc id))
        (href (string-append "/post/" (number->string id))))
    (printf "<li><time>~a</time> <a href=~s>~a</a></li>"
            date href title)))

(define (render-search-result dbc word)
  (printf "<hr>")
  (let ((result (word->posts dbc word)))
    (if (null? result)
        (printf "没有符合条件的结果！")
        (begin
          (printf "<div class=\"postList\"><ul>")
          (let iter ((ret result))
            (unless (null? ret)
                    (single-item dbc (car ret))
                    (iter (cdr ret))))
          (printf "</ul></div>")))))



;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; 会话管理 ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (gen-key uid sec)
  (string-append
   (encrypt (string-append (number->string uid)
                           (cgi-remote-addr)
                           (cgi-http-user-agent)
                           (number->string sec)))
   (number->string (- (* uid 2512) 3))))     ;;uid x 2512 - 3 附在密文后面

(define (set-cookies dbc uid)
  (let ((now (current-seconds)))
    (let ((key (gen-key uid now)))
      (printf "Set-Cookie:CGISESSION=~a;PATH=/;domain=localhost\n" key)
      (create-session uid key now))))

(define (create-session uid key sec)
  (let ((session-file (string-append "sessions/session_" (number->string uid))))
    (call-with-output-file session-file
      (lambda (op)
        (fprintf op "key ~a\n" key)
        (fprintf op "last-login ~a\n" sec))
      #:exists 'replace)))

(define (parse-session uid)
  (let ((session-file (string-append "sessions/session_" (number->string uid))))
    (call-with-input-file session-file
      (lambda (ip)
        (let iter ((line (read-line ip)) (ret '()))
          (if (eof-object? line) ret
              (iter (read-line ip) (cons (string-split line) ret))))))))

(define (update-session uid)
  (let ((now (current-seconds))
        (old-session (parse-session uid)))
    (let ((key (cadr (assoc "key" old-session)))
          (last-login (cadr (assoc "last-login" old-session))))
      (let ((session-file (string-append "sessions/session_" (number->string uid))))
        (call-with-output-file session-file
          (lambda (op)
            (fprintf op "key ~a\n" key)
            (fprintf op "last-login ~a\n" last-login)
            (fprintf op "last-active ~a\n" now))
          #:exists 'replace)))))

;;;;;;;;;;;;;;;;;;;;;;;;;;;; 数据库 ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (initialize-blog! db-file)
  (define dbc (sqlite3-connect #:database db-file #:mode 'create))
  (unless (table-exists? dbc "posts")
          (query-exec
           dbc
           (string-append
            "CREATE TABLE posts "
            "(id integer primary key, "
            "title text, "
            "body text, "
            "date text, "
            "uid integer)"))
          (blog-insert-post!
           dbc
           "Hello, World!" "This is my first post" 1))
  
  (unless (table-exists? dbc "comments")
          (query-exec
           dbc
           "create table comments (pid integer, content text, email text)")
          (post-insert-comment!
           dbc
           1 "First comment!" "sample@xxx.com"))
  (unless (table-exists? dbc "users")
          (query-exec
           dbc
           (string-append
            "CREATE TABLE users "
            "(uid integer primary key, "
            "uname TEXT, "
            "dname TEXT, "
            "password TEXT)"))
          (add-user! dbc site-admin admin-display admin-passwd))
  dbc)

(define (word->posts dbc word)
  (let ((word (string-append "%" word "%")))
    (query-list
     dbc
     "SELECT id FROM posts WHERE title like ?" word)))

(define (user->uid dbc uname)
  (query-value
   dbc
   "SELECT uid FROM users WHERE uname = ?" uname))

(define (uid->user dbc uid)
  (query-value
   dbc
   "SELECT uname FROM users WHERE uid = ?" uid))
   
(define (uid->dname dbc uid)
  (query-value
   dbc
   "SELECT dname FROM users WHERE uid = ?" uid))
  
(define (user->password dbc uname)
  (query-value
   dbc
   "SELECT password FROM users WHERE uname = ?" uname))

(define (all-users db)
  (query-list
   db
   "SELECT uname FROM users"))

;;add user
(define (add-user! db uname dname passwd)
  (query-exec
   db
   "INSERT INTO users (uname, dname, password) values (?, ?, ?)"
   uname dname (encrypt passwd)))

;;encoding password
(define (encrypt str)
  (bytes->string/latin-1 (md5 str)))

;;在 blog 中插入 post
(define (blog-insert-post! db title body uid)
  (query-exec
   db
   "insert into posts (title, body, date, uid) values (?, ?, ?, ?)"
   title body (query-value db "select date('now')") uid))

;;在 post 中插入评论
(define (post-insert-comment! db pid content email)
  (query-exec
   db
   "insert into comments (pid, content, email) values (?, ?, ?)"
   pid content email))

;;
(define (all-posts dbc)
  (query-list dbc "select id from posts"))

;;根据 ID 取出 comments 表中所有的评论
(define (id->comments db id)
  (query-list
   db
   "select content from comments where pid = ?"
   id))

(define (id->title db id)
  (query-value
   db
   "select title from posts where id = ?" id))

(define (id->body db id)
  (query-value
   db
   "select body from posts where id = ?" id))

(define (id->writer db id)
  (query-value
   db
   "select uid from posts where id = ?" id))

(define (id->date db id)
  (query-value
   db
   "select date from posts where id = ?" id))

;;;;;;;;;;;;;;;;;;;;;;;;;;数据库初始化;;;;;;;;;;;;;;;;;;;;;;;;

(define BLOG (initialize-blog! db-file))

;;跳转
(define (reload uri)
  (printf "Location: ~a\n\n" uri))


;;;;;;;;;;;;;;;;;;;;;根据 get 的 uri 判定要执行的动作;;;;;;;;;;;;;;;;;;;;;;;

(define (parse-get uri uid)
  (when uid (update-session uid))
  (cond ((string=? uri "/")
         (render-page BLOG uid))
        
        ((regexp-match? #px"^/post/\\d+" uri)
         (let ((pid (string->number (car (regexp-match #px"\\d+" uri)))))
           (let ((posts (all-posts BLOG)))
             (if (member pid posts)
                 (render-post-page BLOG pid uid)
                 (render-404)))))
        
        ((regexp-match? #px"^/page/\\d+" uri)
         (let ((page (string->number (car (regexp-match #px"\\d+" uri)))))
           (render-page BLOG uid page)))

        ((string=? uri "/search")
         (render-search-head BLOG uid)
         (render-page-footer)
         (page-end))
        
        ((regexp-match? #px"^/search\\?title=" uri)
         (let* ((arg (cadr (string-split uri "=")))
                (word (string-utf-8 (string->bytes/latin-1 arg))))
           (render-search-head BLOG uid)
           (render-search-result BLOG word)
           (render-page-footer)
           (page-end)))
        
        ((string=? uri "/login")
         (render-login-page BLOG))

        ((string=? uri "/logout")
         (when uid
               (logout uid))
         (reload "/"))
        
        (else
         (render-404))))

         
;;;;;;;;;;;;;;;;;;;;;;;;;;;;; post 数据分析 ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

(define (hex->bytes str)
  (let ((hex (subbytes str 1)))
    (bytes (string->number (bytes->string/latin-1 hex) 16))))

;;按照指定的 ascii 码分割 bytes-string
(define (bytes-split bstr n)
  (let ((reg (byte-regexp (bytes n))))
    (regexp-split reg bstr)))

(define (decoding-string byte-str)
  (let ((replaced
         (regexp-replaces
          byte-str
          `((#px#"\\+" ,(bytes 32))                  ;替换 + 号
            ;(#rx#"%0[Dd]%0[Aa]" #"</p><p>")          ;替换回车
            ;(#rx#"%3[Cc]" #"%26lt;")                 ; <
            ;(#rx#"%3[Ee]" #"%26gt;")                 ; >
            (#px#"%[A-Fa-f0-9]{2}" ,hex->bytes)))))
                                   ;% 开头的十六进制数交给hex->bytes处理
    replaced))

;;输入：初始的 URL bytes
;;输出：最终的字符串，编码为 UTF-8
(define (string-utf-8 byte-str)
  (if (equal? byte-str #"") ""
      (bytes->string/utf-8 (decoding-string byte-str))))


;;将 post 上来的数据先按 & 分割成一个列表
;;再在列表元素上按 = 号分割成嵌套列表
(define (parse-post-data)
  (let ((input-data (port->bytes)))
    (let* ((split-1 (bytes-split input-data 38))  ; ascii 38 = '&'
           (split-2 (map (lambda (b)
                           (bytes-split b 61))    ; ascii 61 = '='
                         split-1)))
      split-2)))

;;添加一条 post
(define (add-post uid)
  (let ((post-str (parse-post-data)))
    (let ((title (string-utf-8 (cadr (assoc #"title" post-str))))
          (body
           (string-append "<p>"
                          (string-utf-8 (cadr (assoc #"body" post-str)))
                          "</p>")))
      (blog-insert-post! BLOG title body uid)
      (reload "/"))))

(define (login)
  (let ((post-str (parse-post-data)))
    (let ((uname (string-utf-8 (cadr (assoc #"uname" post-str))))
          (passwd (string-utf-8 (cadr (assoc #"passwd" post-str)))))
      (let ((users (all-users BLOG)))
        (if (not (member uname users))
            (printf "User not exists!") ;;todo: http-header
            (let ((p (user->password BLOG uname))
                  (input-p (encrypt passwd)))
              (if (string=? p input-p)
                  (begin
                    ;;(printf "Content-type: text/html\n\n")
                    (set-cookies BLOG (user->uid BLOG uname))
                    (reload "/"))
                  (begin
                    (printf "User Name or Password wrong!<br><br>")
                    ))))))))

(define (logout uid)
  (let ((session-file (string-append "sessions/session_" (number->string uid))))
    (delete-file session-file)))

;;在 post 下添加一条评论
(define (add-comment uri)
  (let ((pid (string->number
              (last (string-split uri "/")))))    
    (let ((post-data (parse-post-data)))
      (let ((email (string-utf-8 (cadr (assoc #"email" post-data))))
            (content (string-utf-8 (cadr (assoc #"content" post-data)))))
        (post-insert-comment! BLOG pid content email)        
        (reload
         (string-append "/post/" (number->string pid)))))))

(define (parse-post uri uid)
  (cond ((string=? uri "/add-post")
         (if uid
             (add-post uid)
             (reload "/login")))
        ((regexp-match? #px"^/add-comment/\\d+$" uri)
         (add-comment uri))
        ((string=? uri "/login")
         (login))
        (else
         (printf "Status: 400 Bad Request\n\n")
         (printf "HTTP 400 - Bad Request"))))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;                          read cookies
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


(define (read-cookie)
  (let ((cook (cgi-http-cookie)))
    (if cook
        (string-split cook "=")
        #f)))

(define (get-uid)
  (let ((cookie (read-cookie)))
    (if cookie
        ;; (if (string=? (car cookie) "CGISESSION")
        ;; (if (> (string-length (cadr cookie)) 32)
        (let ((uid-txt (substring (cadr cookie) 32)))
          (let ((uid (/ (+ (string->number uid-txt) 3) 2512)))
            (if (file-exists? (string-append "sessions/session_" (number->string uid)))
                (let ((last-session (parse-session uid)))
                  (if (string=? (cadr (assoc "key" last-session))
                                (cadr cookie))
                      (let ((time1 (or (assoc "last-active" last-session)
                                       (assoc "last-login" last-session))))
                        (let ((time2 (string->number (cadr time1))))
                          (if (< (- (current-seconds) time2) timeout)
                              uid
                              #f))) ;超时
                      #f))          ;key被修改
                #f)))                ;硬盘上的 session_id 文件不存在
        ;; #f)  ;key 短于32
        ;; #f)  ;COOKIES NAME 不等于 CGISESSION
        #f)))                             ;cookies 不存在

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;                          启动CGI监听
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
(cgi #:request
     (lambda ()
       (let ((uri (cgi-request-uri)) (uid (get-uid)))
         (cond ((equal? (cgi-request-method) "POST")
                (parse-post uri uid))
               ((equal? (cgi-request-method) "GET")
                (parse-get uri uid))
               (else
                (printf "Status: 400 Bad Request\n\n")
                (printf "HTTP 400 - Bad Request"))))))
```

对应的 CSS 布局：

```css
*{
    padding:0;
    margin: 0;
}

body{
    font-size:1em;
    letter-spacing:0.5px;
    padding:0 10px 0 10px;
    margin-left:auto;
    margin-right:auto;
    font-family:Verdana,Arial,"宋体";
    /*font-size: 12px;*/
    line-height: 1.5;
}

hr {
    margin: 5px 10px 5px;
    height:1px;
    border:none;
    border-top:1px solid #555555;
}

p {
    padding:5px 0 5px 0;
    margin:5px 0 5px 0;
}

textarea{
    width: 70%;
    height: 180px;
    padding:5px;
    margin: 5px;
}

pre {
    font-size:0.9em;
    margin: 10px 0px 0px;
    padding: 10px;
    border: 1px dotted #785;
    background: none repeat scroll 0% 0% #F5F5F5;
}

form {
    padding: 20px;
}

.short {
    width: 40%;
    padding:5px;
    margin: 5px;
}

time {
    font-size:11px;
}

ul,li {
    list-style-type:none;
}

#main{
    width:80%;
    margin:auto;
    display:block;
    background: none repeat scroll 0% 0% #F7F7F7;
    box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.5);
    overflow: hidden;
    border-left: 1px solid #CACACA;
    border-right: 1px solid #CACACA;
}
/***************************** banner ******************************/
#banner {
    width: 100%;
    height:100px;
    box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.5);
    background-color: #EFEFEF;
}
#site-title {
    margin: 10px;
    width:40%;float:left;
}
#entrance {
    position: relative;
    width:20%;
    float:right;
    margin: 10px;
    text-align:right;
    font-size: 12px;
}

/*************************** menu ********************************/
#menu-items {
    float:left;
    width:100%;
    padding:0;
    margin:0;
    list-style-type:none;
}

.menu-item  a{
    text-align:center;
    float:right;
    width:4em;
    text-decoration:none;
    color:black;
    background-color:BurlyWood;
    padding:0.2em 0.6em;
    border-top-left-radius:8px;
    border-top-right-radius:8px;

    border-left:1px solid white;
    border-top:1px solid white;
    border-right:1px solid white;
}

.menu-item {
    display: inline;
}


/*************************** post list ****************************/
.postList {
    position: relative;
    top: 10px;
    width: 100%;
    margin: 20px;
}

#pager {
    font-size:12px;
}
/*************************** footer *******************************/
#footer {
    text-align:center;
    font-size:0.8em;
}
/*************************** single post page *********************/
#post-title {
    width:100%;
    float:left;
    text-align:center;
    font-size:1em;
    margin: 20px 0 10px 0;
}

#post-info {
    width:100%;
    float:left;
    text-align:center;
    font-size: 11px;
    margin: 10px 0 10px 0;
}

#post-body {
    padding: 10px;
    float:left;
}

.clear {
    clear: both;
}

#comments {
    padding:5px;
    margin: 5px;
}

.comment {
    padding:5px;
    margin: 5px;
    background-color:#e6e6e6;
    border-radius:10px;
}

/************************* Login ****************************/

#login {
    position: relative;
    top: 10px;
    width:400px;
    height:200px;
    margin-left:auto;
    margin-right:auto;
    border-style: solid; border-width: 1px;
    background-color:#e6e6e6;
}

.login-l {
    float:left;
    margin: 20px;
}    

.login-r {
    float:right;
    margin: 18px;
}    

/*************************** search ***************************/
.blank {
    width: 100%;
    height: 20px;
}
```

Nginx 配置文件：

```
server {
    location / {
        root      /site-root-here/;
        include   scgi_params;
        scgi_pass localhost:4000;
    }

    location ~* \.(gif|jpg|jpeg|png|css|js|ico|html)$ {
        root      /site-root-here/;
    }  
}
```

Nginx 已经内置了 SCGI 支持。除了图片、静态 html、jc、css代码，Nginx 会将所有的请求转发到后端的 SCGI SERVER.