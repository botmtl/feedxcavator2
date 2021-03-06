# feedxcavator2

This thing is able to convert anything to RSS with an arbitrary level of 
fine-tuning by using CSS selectors. There is a built-in [pubsubhubbub](https://en.wikipedia.org/wiki/PubSubHubbub)
server, and it's also possible to program an arbitrary
feed-extraction process in a simple Clojure-based [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) directly through the web ui.
Because it's designed as a Google App Engine application, it's troublesome to 
create a GAE accounts, upload GAE applications and analyze web pages manually to 
craft necessary CSS selectors, probably no one would use the app except me, 
so here is what it looks like:

[DOWNLOAD](https://github.com/GChristensen/feedxcavator2/releases/download/2.0.0/feedxcavator-2.0.0.zip) :: [VIDEO MANUAL](https://youtu.be/jHKo4CM-Qfw)

<a href="https://github.com/GChristensen/feedxcavator2/blob/master/img/xcavator.png" target="_blank"><img src="https://github.com/GChristensen/feedxcavator2/blob/master/img/xcavator_thumb.png" /></a>&nbsp;&nbsp;<a href="https://github.com/GChristensen/feedxcavator2/blob/master/img/custom.png" target="_blank"><img src="https://github.com/GChristensen/feedxcavator2/blob/master/img/custom_thumb.png" /></a>

tl&dr: this is an RSS producer that is able to transform any site
to RSS (IT'S NOT AN RSS AGGREGATOR!).

### Supported CSS Subset

Only the following CSS capabilities are currently supported by __feedxcavator2__:

<pre>
* Elements:                     div
* IDs:                          div#some-id
* Classes:                      div.some-class
* Descendants:                  h1 a
* Direct hierarchy:             div#some-id > h1.some-class > a
* Attribute check:              a[attr]
* Attribute value:              a[attr="value"]
* Attribute substring:          a[attr*="substr"]
* Pseudo-classes:               h1:first-child
* Parameterized pseudo-classes: a:nth-of-type(3)
</pre>

__feedxcavator2__ uses [enlive](https://github.com/cgrand/enlive#readme)
library for HTML processing and internally converts all CSS selectors into
[enlive selectors](http://enlive.cgrand.net/syntax.html).
The conversion routine is quite straightforward, so it's 
better to use enlive selectors in complex cases if css selectors do not work. 
__feedxcavator2__ will assume that elnive selectors are used if the selector 
string is wrapped in square brackets (e.g. [:div#some-id :> :h1.some-class 
:> :a] or [root]) and will not try to convert them.
Although, regular CSS selectors should successfully deal with relatively simple hierarchical 
queries, which should be enough in the majority of cases.

### Ordinary and custom extractors

__feedxcavator2__ will use only supplied CSS selectors to extract verbatim data if the 
"Custom excavator" field is left blank. If not, it assumes that the field contains the name 
of a custom extractor function defined in Clojure programming language at the "Custom 
extractors" page. In this case field "Custom parameters" may contain a string-readable 
Clojure datum which will be fed to `read-string` function and passed to the extractor.

There are two types of processing functions which could be defined by the `defextractor` and
`defbackground` macros. The first is intended for direct and fast data conversion,
the later should be used for heavy feeds fetched in the background. Functions defined
with `defextractor` are called during the direct feed URL request (processing time of 
foreground GAE instance requests is limited to one minute), functions defined with
`defbackground` are called in background tasks defined by `deftask` macro (the tasks 
are executed by GAE backend instances with no time limit). 
For the feeds with `defbackground` extractors feed link request will return RSS stored earlier by 
the task (you don't need anyhow bother on this or specify this in feed settings, 
all is resolved by DSL), so RSS will change only after the next task execution. 

Background task will also automatically notify aggregator through pubsubhubbub protocol 
if the "Realtime" flag is checked in the feed settings, so the aggregator can get data of realtime
feeds just after extraction.

Extraction DSL example:

```clojure
;; task setup 

;; define tasks that will fetch all feeds which name (specified in the "Feed title" 
;; field at the task settings) contain one of the given strings
;; extractors of these fields should be defined with `defbackground` macro
(deftask fetch-daily-feeds ["autofetch"])
(schedule fetch-daily-feeds 13 00) ; GMT
(schedule fetch-daily-feeds 18 00)

(deftask fetch-periodic-feeds ["tumblr:mass" "fb:" "good morning"])
(schedule fetch-periodic-feeds 13 10)

;; utility functions

;; transform page defined by url to a list of the maps (headlines) with the following fieleds:
;; {
;;  :title "headline title" 
;;  :link "headline url" 
;;  :summary "article summary" 
;;  :image "image url" 
;;  :html <enlive headline html representation>
;; }
;; api/apply-selectors function magically knows how to apply selectors from the feed settings 
;; to the page and transform the given parsed elinve html representation to the list of headlines 
;; described above
;; it's also possible to make enlive selects from the :html field which may be necessary, 
;; for example, when some data should be extracted from `style` html attribute, etc.
(defn parse-page [url]
  (let [response (api/fetch-url url)
        doc-tree (api/resp->enlive response)]
    (when doc-tree
      (api/apply-selectors doc-tree))))
 
;; filter out headlines which links are already has been seen by the fetcher 
(defn filter-fetcher-history [uuid headlines]
  (let [history (:entries (db/query-fetcher-history uuid))
        result (filter #(not (history (:link %))) headlines)]
    (db/store-fetcher-history! uuid (map #(:link %) headlines))
    result))

;; extractors

;; `feed-settings` parameter contains data from feed settings, `params` hold the value of the
;; "Custom parameters" field passed through `read-string` function
;;
;; an extractor should return a collection of headline maps with the following fields:
;; {
;;  :title "headline title" 
;;  :link "headline url" 
;;  :summary "article summary" 
;;  :image "image url" 
;; }
;; all other fields are ignored 
(defextractor multipage-extractor [feed-settings params]
  (apply concat (parse-page (:target-url feed-settings)) ; URL from the "Target URL" field
                (parse-page (str (:target-url feed-settings) "/page/2"))))

;; fetch some threads from a set of forums of the Bulletin Board; the "Custom parameters" field 
;; shoud contain forum numeric ids in the form of the following text:
;; [1 2 3]
;; which will be converted to Clojure vector by read-string
(defbackground bb-extractor [feed-settings params]
  (apply concat
         (for [forum params]
           (let [forum-url (str (:target-url feed-settings) forum)
                 ;; stage 1: extract thread URLs from forum pages (the corresponding selectors 
                 ;; should be specified in feed settings) and filter out already seen urls
                 threads (filter-fetcher-history forum-url (parse-page forum-url))                                             
             (when (not (empty? threads))
               ;; stage 2: fetch thread pages and extract data from first posts using enlive
               (for [t threads]
                 (let [trhead (api/fetch-url (:link t))
                       thread-tree (when thread (api/resp->enlive thread))]
                   (assoc t
                     ;; remove html tags and special character entities
                     :title (api/untag (api/html-unescape (:title t)))
                     ;; get content of the src field of the first <img> tag from post text
                     :image (:src (:attrs (first (select thread-tree [:.post_text :img]))))
                     ;; get html string representation of the first tag with .post_text class 
                     :summary (api/render (first (select thread-tree [:.post_text])))))))))))

;; extract data from json api
;; "Custom parameters" field hould contain owner id in the form of following text:
;; 123
;; which will be embedded into api url
(defbackground json-extractor [feed-settings params]
    (let [api-token "..."
          api-version "1"
          url (str "https://json.api/method/data.get?owner_id=" params 
                   "&access_token=" api-token "&v=" api-version)
          response (api/fetch-url url)
          content (api/resp->str response)
          posts (((json/read-str content) "response") "items")
          headlines (for [p posts]
                      {
                       :title (p "title")
                       :link (p "url")
                       :summary (p "text")
                       :image (or ((p "attachments") "photo_640")
                                  ((p "attachments") "photo_320"))
                      })]
      (sort-by :link #(compare %2 %1) headlines)))
      
;; transform another rss (just turn titles upper case)
(defextractor rss-extractor [feed-settings params]
  (let [response (api/fetch-url (:target-url feed-settings))
        doc-tree (api/resp->enlive-xml response)]
    (for [i (select doc-tree [:item])]
      (let [tag-content #(apply str (:content (first (select i [%]))))]
         {
          :title (str/upper-case (tag-content :title))
          :link (tag-content :link)
          :summary (tag-content :description)
         })))))
```

### Private Deployment

You may [install](http://code.google.com/appengine/docs/java/gettingstarted/uploading.html) 
a private [instance](https://github.com/GChristensen/feedxcavator2/releases/download/2.0.0/feedxcavator-2.0.0.zip)
of the application on your GAE account. Оnly the account owner will be able 
to create or manage feeds (but still be able to share feed links). The only 
thing you need to do is to fill-in application id in the 'appengine-web.xml' file.

### Working on the code

To compile the project you need to install [this](https://github.com/GChristensen/appengine-magic) fork of 
appengine-magic into your local leiningen repository (yes, appengine-magic still works in 2017).

### License

Copyright (C) 2017 g/christensen (gchristnsn@gmail.com)

Distributed under the Eclipse Public License, the same as Clojure.

