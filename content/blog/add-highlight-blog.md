+++
title = "add highlight blog"
draft = false 
date = "2016-11-24T11:41:49-05:00"

+++

Please note I have changed my blogging engine to [hugo](https://gohugo.io/)

### Adding highlight.js to the Blog

Per the community the easist way to get javascript libraries into your project is to use [cljsjs](http://cljsjs.github.io/). However I tired to use it and could not get it to work quickly. I looked at the code and saw that it is basically downloading a external zip. It looked too complicated for what it was doing. I decided to go with externs instead.

The basic trick is to integrate the highlighting in the React life cycle. Luckily there is a reagent cookbook recipe on this already. It can be found [here](https://github.com/reagent-project/reagent-cookbook/tree/master/recipes/markdown-editor). I took the recipe and simplified it a little.

First the project.clj file needs to be updated.

Add dommy in the dependencies:

```clojure
[prismatic/dommy "1.1.0"]
```

The :cljsbuild map will need to be updated with :externs. Mine looks like this:

```clojure
:cljsbuild {:builds {:app {:source-paths ["src/cljs" "src/cljc"]
                           :compiler {:output-to "target/cljsbuild/public/js/app.js"
                                      :output-dir "target/cljsbuild/public/js/out"
                                      :asset-path   "js/out"
                                      :optimizations :none
                                      :pretty-print  true
                                      :externs ["externs/syntax.js"]}}}}
```

Add a extern file: externs/syntax.js with the following contents:

```javascript

var hljs = {};
hljs.highlight = function (name, value, ignore_illegals, continuation) {};
hljs.highlightAuto = function (text, languageSubset) {};
hljs.fixMarkup = function (value) {};
hljs.highlightBlock = function (block) {};
hljs.configure = function (user_options) {};
hljs.initHighlighting = function () {};
hljs.initHighlightingOnLoad = function () {};
hljs.registerLanguage = function (name, language) {};
hljs.listLanguages = function () {};
hljs.getLanguage = function (name) {};
hljs.inherit = function (parent, obj) {};

```

The original recipe used this function to highlight code:

```clojure
(defn highlight-code [html-node]
  (let [nodes (.querySelectorAll html-node "pre code")]
    (loop [i (.-length nodes)]
      (when-not (neg? i)
        (when-let [item (.item nodes i)]
          (.highlightBlock js/hljs item))
        (recur (dec i))))))
```

This can be simplified by using the [dommy](https://github.com/plumatic/dommy) library. Here is the new fn:

```clojure
(defn highlight-code [html-node]
  (let [nodes (sel [:pre :code])]
    (doall (for [node nodes]
               (.highlightBlock js/hljs node)))))
```

Then the projects existing markdown-component fn will change from:

``` clojure
(defn markdown-component [content]
    [:div {:dangerouslySetInnerHTML
           {:__html (-> content md->html)}
```

to this:

```clojure
(defn markdown-component [content]
           [(with-meta
              (fn []
                [:div {:dangerouslySetInnerHTML
                       {:__html (-> content md->html)}}])
              {:component-did-mount
                (fn [this]
                  (let [node (reagent/dom-node this)]
                    (highlight-code node)))})])
```

Last step is to update the handler.clj file to load the highlight.js css and js files.

``` clojure
(def loading-page
  (html
   [:html
    [:head
     [:meta {:charset "utf-8"}]
     [:meta {:name "viewport"
             :content "width=device-width, initial-scale=1"}]
     (include-css "//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.1.0/styles/agate.min.css"
                  (if (env :dev) "css/tufte.css" "css/tufte.min.css"))]
    [:body
     mount-target
     (include-js
      "//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.1.0/highlight.min.js"
      "js/app.js")]]))
```

Note that the highlight css file has different styles available. The styles can be found on the highlight.js [demo page](https://highlightjs.org/static/demo/). 
