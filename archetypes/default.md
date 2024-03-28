+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
discription = ""
tags = []
categories = ["{{ path.Base .File.Dir }}"]
date = {{ .Date }}
draft = false
slug = ""
+++
