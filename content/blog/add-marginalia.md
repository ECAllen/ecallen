+++
title = "add marginalia"
draft = false 
date = "2016-11-22T11:41:49-05:00"

+++

Please note I have changed my blogging engine to [hugo](https://gohugo.io/)

### Adding marginalia pages to the blog

Adding Marginalia pages is dead simple using the [lein-marginalia](https://github.com/MichaelBlume/marginalia) plug-in. Make sure you use Michael Blume's [fork](https://github.com/MichaelBlume/lein-marginalia) as the original project seems to have become inactive.

So let's create some documentation. From the projects root directory generate the doc:

```shell
bright_paper_werewolves (master)> lein marg -d resources/public/ -f blog-marg.html
```

That will create a blog-marg.html file in the resources/public dir. The file will be self contained with all the javascript and css needed.

Now the file needs to be served. Because of [Compojure's](https://weavejester.github.io/compojure/compojure.route.html#var-resources) "resources" route in the handler.clj anything in resources/public will automatically be served up.

```clojure
(defroutes routes
  ...
  ;; resources will serve anything in resources/public
  (resources "/")
  (not-found "Not Found"))
```

The (resources "/") line is part of the reagent template.

The last step is to add the link to the menu.

```clojure
(defn menu-link [url txt]
  [:a {:href url :class "menu"} txt])

(defn top-menu []
    [:ul {:class "menu"}
     [:li {:class "menu"}
      (menu-link "/" "Blog")
      ;; this links to the blog-marg.html file in resources/public
      (menu-link "/blog-marg.html" "Marginalia")]])
```
