+++
title = 'Go OTel Logging'
date = 2025-02-08T12:42:36+01:00
draft = true
+++

# OTel logging in Go

To start with let's quickly review the conceptual flow of logs from the creation (we'll be using `slog` package) to the moment they're sent to a collector or a backend.
