---
layout: default
title: Open Source Projects
keywords: 开源,open-source,GitHub,开源项目
description: 开源改变世界。
permalink: /open-source/
---

{% assign sorted_repos = site.data.contributions %}

<section class="container">
    <header class="text-center">
        <h1>Open Source Projects</h1>
        Projects on Github
    </header>
    <div class="repo-list">
        <!-- Check here for github metadata -->
        <!-- https://help.github.com/articles/repository-metadata-on-github-pages/ -->
        {% for repo in sorted_repos %}
        <a href="{{ repo.url }}" target="_blank" class="one-third-column card text-center">
            <div class="thumbnail">
                <div class="card-image geopattern" data-pattern-id="{{ repo.name }}">
                    <div class="card-image-cell">
                        <h3 class="card-title">
                            {{ repo.name }}
                        </h3>
                    </div>
                </div>
                <div class="caption">
                    <div class="card-description">
                        <p class="card-text">{{ repo.descr }}</p>
                    </div>
                    <div class="card-text">
                        <span class="card-text" title="Role：{{ repo.role }}">
                            <span class="octicon octicon-person"></span>
                            <p class="card-text">{{ repo.role }}</p>
                        </span>
                    </div>
                </div>
            </div>
        </a>
        {% endfor %}
    </div>
</section>
