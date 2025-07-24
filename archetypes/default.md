+++
date = '{{ .Date }}'
tags = ["tag"]
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
draft = true
+++