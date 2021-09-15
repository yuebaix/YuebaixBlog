---
title:       {{ $prefix := slicestr .Date 0 10 }}{{ $prefix := printf "%s%s" $prefix "-" }}{{ $title := replace .Name $prefix "" }}{{ $title := replace $title "_" " " | title }}"{{ $title | title }}"
subtitle:    ""
description: ""
author:      "月白"
date:        {{ .Date }}
URL:         ""
image:       ""
categories:  ["TECH"]
tags:        ["干货"]
---

