---
layout: post
title: "Clojurescript and nodejs"
date: 2015-02-13T16:49:49+00:00
comments: true
---


Here's a little snippet for driving express from clojurescript (run in cljs repl):

	(ns express_sample
	  (:require [cljs.nodejs :as node]))

	(def express (node/require "express"))
	(def app (express))

	(defn -main [& args]
	  (doto app
	    (.get "/" (fn [req res]
	                (.send res "Hello World")))
	    (.listen 3000)))

	(-main)

