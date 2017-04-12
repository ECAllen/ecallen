+++
draft = false 
title = "clojurescript blog in 10 minutes"
date = "2016-06-22T11:41:49-05:00"

+++

Please note I have changed my blogging engine to [hugo](https://gohugo.io/)

### The Challenge

I was listening to an excellent [podcast with Eric Normand](http://giantrobots.fm/172)  where he mentioned that the tooling around clojurescript was mature enough to recreate the famous [Ruby 15 minute blog engine](https://www.youtube.com/watch?v=Gzj723LkRJY). So I felt like mucking about with clojurescript and I was tired of my previous blog platform. Since I have been playing with react/reagent on other projects I decided to see how long it would take before I could get something useful up and running.

I will create two projects: one for the fronted SPA (Single Page App) and another that will simply serve up blog posts using a data structure. Normally there would be one project to do both but I decided to separate them into two projects.

#### The Solution

##### Shape of the data

First we need to decide on the shape of data structure that the SPA and the back-end  will share. The following should do:

```clojure
(def blog-posts
  {:ecallen
   {:UUID
    {:title "Bright Paper Werewolves"
     :author "ECAllen"
     :post-timestamp "some epoch time"
     :revision 0
     :last-update "some epoch time"
     :text "..." }}})
```
### Backend/Liberator REST server

Now that we know what the data will look like, let's create a process to serve it up to the front end. At first it will only respond to a GET and deliver a simple map via transit using liberator.

Create the blog server using lein

```shell
lein new blog-server
```

Add the needed plugins and libs to the project.clj.

```clojure
(defproject blog-server "0.1.0-SNAPSHOT"
  :description "server for blog posts"
  :url "https://github.com/ECAllen/blog-server.git"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.7.0"]
                 [liberator "0.14.0"]
                 [compojure "1.4.0"]
                 [io.clojure/liberator-transit "0.3.0"]
                 [ring-transit "0.1.4"]
                 [liberator "0.14.0"]
                 [ring-cors "0.1.7"]
                 [ring/ring-core "1.4.0"]
                 [ring/ring-jetty-adapter "1.4.0"]
                 [juxt/dirwatch "0.2.3"]
                 [environ "1.0.2"]]
  :ring {:handler blog-server.core/handler}
  :main blog-server.core
  :uberjar-name "blog-server.jar"
  :profiles {:uberjar {:env {:production true}
                       :aot :all
                       :omit-source true}}
 :plugins [[lein-ring "0.9.7"]
           [lein-environ "1.0.2"]])
```

Next code needs to be written to respond to a GET request and deliver the map to the browser. This is the minimal code to get that done.

```clojure
(ns blog-server.core
  (:gen-class)
  (:require
   [liberator.core :as liber]
   [clojure.pprint :as pp]
   [compojure.core :as compj]
   [compojure.route :as route]
   [compojure.handler :as handler]
   [compojure.response :as response]
   [ring.middleware.params :as param]
   [ring.middleware.cors :as cors]
   [io.clojure.liberator-transit :as transit]
   [ring.middleware.transit :as ring-transit]
   [ring.adapter.jetty :refer [run-jetty]]
   [clojure.string :as str]
   [clojure.edn :as edn]
   [juxt.dirwatch :refer [watch-dir]]
   [environ.core :refer [env]])
  (:import java.io.File))


;; ======================
;; Globals
;; ======================

(def blog-file (env :blog-file))
(def blog-dir (env :blog-dir))
(def port (Integer/parseInt (env :port)))

;; ======================
;; Persist
;; ======================

(defn read-file [file]
  (edn/read-string (slurp file)))

(defn save-file [data file append-mode]
  (with-open [f (clojure.java.io/writer file :append append-mode)]
    (pp/pprint @data f)))

(defn get-posts [file]
  (if (not (.exists (File. file)))
    (save-file (atom {}) file true))
  (atom (read-file file)))

(def blog-posts (get-posts blog-file))

(defn update-posts [event]
  (let [file (-> event :file bean :path str)
        action (:action event)]
    (if (and (= action :modify) (= file blog-file))
      (do
        ; (prn "debug" event)
        (reset! blog-posts (read-file blog-file))))))

;; ======================
;; Resources
;; ======================
(liber/defresource posts [userid]
  :available-media-types ["application/transit+json"
                          "application/transit+msgpack"
                          "application/json"]
  :handle-ok (fn [_] (get @blog-posts (keyword userid))))

;; ======================
;; Routes/Hndlers
;; ======================

(compj/defroutes main-routes
  (compj/GET "/posts/:userid" [userid] (posts userid))
  (route/not-found "<h1>Page not found</h1>"))

(def handler
  (-> main-routes
      param/wrap-params
      ring-transit/wrap-transit-params
      (cors/wrap-cors :access-control-allow-origin [#".*"]
                      :access-control-allow-methods [:get :post :options]
                      :access-control-allow-headers ["content-type"])))

;; ======================
;; Main
;; ======================

(defn -main [ & args]
  (run-jetty handler {:port port :join? false})
  (watch-dir update-posts (clojure.java.io/file blog-dir)))

```
The above code watches a file which holds the map for the blog post's data. When the file is updated the atom that holds the blog posts will be updated. Yes,most solutions would use a databases here, but that is overkill for my purposes. This is only a personal blog.

OK now that's out of the way. Let's switch back to the front end and finish the minimum viable code there.

### Frontend/React based SPA (Single Page App)

First create a new reagent project named after a song from one of [my favorite bands](http://genius.com/Guided-by-voices-bright-paper-werewolves-lyrics):

```shell
> lein new reagent bright-paper-werewolves
```

add the following libs to the project.clj file

```clojure
[cljs-ajax "0.5.1"]
[markdown-clj "0.9.85"]
[com.cognitect/transit-cljs "0.8.232"]
```

Now for the code. Most of the code below came from the lein reagent template.

```clojure
(ns bright-paper-werewolves.core
    (:require [reagent.core :as reagent]
              [reagent.ratom :as ratom]
              [reagent.debug :as debug]
              [reagent.session :as session]
              [secretary.core :as secretary :include-macros true]
              [accountant.core :as accountant]
              [ajax.core :as ajax]
              [markdown.core :refer [md->html]]
              [cognitect.transit :as transit]))


;; -------------------------
;; Globals

;; reagent vars for react
(def posts (reagent/atom {}))
(def p (ratom/run! (debug/println "posts: " @posts)))

;; blog-server info, for now assume static
;; --------------------------
;; Testing
;; (def server "localhost")
;; (def port "3000")

;; --------------------------
;; Production
(def server "ecallen.com")
(def port "4010")

(def userid "ecallen")

;; -------------------------
;;  AJAX

;; error handler for debugging
(defn error-handler [response]
  (debug/println (str "Error status: " (:status response)))
  (debug/println (str "Status details: " (:status-text response)))
  (debug/println (str "Failure: " (:failure response)))
  (if (contains? response :parse)
    (debug/println (str "Parse error: " (:parse response)))
    (debug/println (str "Original Text: " (:original-text response))))
  (debug/println (str "Error response: " response)))

(defn posts-handler [response]
  (reset! posts {})
  (swap! posts conj response))

(defn ajax-get [url handler error-handler]
  (ajax/GET url
    {:handler posts-handler
     :error-handler error-handler
     :response-format :transit}))

(defn get-posts []
  (ajax-get (str "http://" server ":" port "/posts/" userid) posts-handler error-handler))

;; -------------------------
;; Views

(defn markdown-component [content]
    [:div {:dangerouslySetInnerHTML
           {:__html (-> content md->html)}}])

(defn home-page []
  [:div
   [:h1 "ECAllen"]
   (doall (for [k (keys (sort-by (comp :post-timestamp second) > @posts))]
           ^{:key k}
           [:section
             [:h2 (get-in @posts [k :title])]
             [:p (get-in @posts [k :post-timestamp])]
             [:p (markdown-component (get-in @posts [k :text]))]]))])


(defn current-page []
  [:div [(session/get :current-page)]])

;; -------------------------
;; Routes

(secretary/defroute "/" []
  (session/put! :current-page #'home-page))

;; -------------------------
;; Initialize app
(get-posts)

(defn mount-root []
  (reagent/render [current-page] (.getElementById js/document "app")))

(defn init! []
  (accountant/configure-navigation!)
  (accountant/dispatch-current!)
  (mount-root))
```

### Style the page

We are going to make it look nice by adding [tufte.css]( https://edwardtufte.github.io/tufte-css/)

Clone the tufte repo then copy the needed files to the clojurescript project

```shell
cp tufte.css ~/projects/bright_paper_werewolves/resources/public/css/
cp -r et-book ~/projects/bright_paper_werewolves/resources/public/css/
```
Edit the hander file to use the new css file src/clj/bright_paper_werewolves/handler.clj

Make the include-css line look like the following:

```clojure
(include-css (if (env :dev) "css/tufte.css" "css/tufte.css"))
```

### Adding blog posts

For now I am keeping the posts in a file under git. When I want to CRUD posts I simply update the file and then do the git pull on the server. The blog server will watch the file and update the posts when it is modified. This is admittedly a very idiomatic process and it works for a personal blog.

### Conclusions

Was I able to do this in 15 minutes?

No. But I was able to do about 90% of it in 15 min. The remaining 10% took quite a bit longer but this was because I was using libraries that I had never used before and I was figuring things out. I don't have a lot of web programming experience with clojure so I was surprised at how much I was able to do in a short time.

Is my solution as robust as the Ruby 15 minute blog?

No but it is mainly due to my inexperience and because I was creating something mainly for myself, so I don't care if it is idiomatic to my work-flow.

Do I think a 15 minute clojurescript solution could be created?

Absolutely, but it might take 20 minutes. The main hurdle in the clojure(script) ecosystem is understanding all the libraries. You don't have this friction in the Rails framework but I am certain the equivalent work could be accomplished. Honestly if you created the right lein templates or utilized boot the process could be incredibly streamlined.

### github pages

https://github.com/ECAllen/blog-server

https://github.com/ECAllen/bright_paper_werewolves"
