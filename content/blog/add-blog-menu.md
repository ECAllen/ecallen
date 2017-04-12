+++
draft = false
title = "add blog menu"
date = "2016-06-23T11:41:49-05:00"

+++

Please note I have changed my blogging engine to [hugo](https://gohugo.io/)

### Adding a menu to the blog

OK the blog is kind of plain. Also I would like to have different section. So I am going to add a simple top menu. It is a pretty straight forward process. I think the hardest part is understanding how the various parts of the reagent template interact with each other.   

First create a css file named site.css in the resources/public/css dir:

```css
ul.menu {
    list-style-type: none;
    margin: 0;
    padding: 0;
    overflow: hidden;
    width: 100%;
}

li.menu {
    display: inline;
    float: left;
}

li.menu a.menu {
    text-align: left;
    border-left: 1px solid #bbb;
    padding: 14px;
    text-decoration: none;
}
```
The above is needed because the tufte.css has no styling for a menu.

Then the handler.clj file needs to be updated to pull in this new css. Add the new css file as an arg to the include-css function.

``` clojure
(include-css
 (if (env :dev) "css/tufte.css" "css/tufte.min.css")
  ...
 "css/site.css")
```

These two functions will actually create the menu and will go into the core.cljs file.

```clojure
(defn menu-link [url txt]
  [:a {:href url :class "menu"} txt])

(defn top-menu []
    [:ul {:class "menu"}
     [:li {:class "menu"}
      ;; add menu items here
      (menu-link "/" "Blog")
      (menu-link "/marginalia" "Marginalia")]])
```

Here is an example of how the function is used to create the top menu for the home page:

```clojure
(defn home-page []
  [:div
    [:div {:class "fullwidth"}
      (top-menu)]
    [:div
      [:h1 "ECAllen"
      ...
```

That's it.
