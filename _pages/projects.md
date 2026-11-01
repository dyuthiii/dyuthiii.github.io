---
layout: archive
title: "Projects"
permalink: /projects/
author_profile: true
---

{% include base_path %}

Independent and course projects with public code. You can find more in my <a href="https://github.com/dyuthiii/Portfolio-Guide/blob/main/README.md" target="_blank">portfolio</a>.

### Snopes Fact-Check Classifier
Scraped Snopes fact-check articles from January to July 2025, then built a TF-IDF and logistic regression classifier to predict whether a claim was rated true or false. The model reached 0.94 accuracy, with an F1 of 0.83 on the smaller "false" class. I also built a Tableau dashboard of article categories and ratings.

*Tools*: Python, Beautiful Soup, scikit-learn, Tableau

<a href="https://github.com/dyuthiii/Snopes" target="_blank">Code</a> | <a href="https://public.tableau.com/views/SnopesTableauViz/Dashboard1" target="_blank">Dashboard</a>

### Political Contributions Database
Used the FEC's API to scrape financial contributions to politicians from January 2022 to November 2024, and stored them in a queryable relational database.

*Tools*: Python, SQL, SQLite

<a href="https://github.com/dyuthiii/PoliticalFinanContributions" target="_blank">Code</a>

### NLP on Medical Company Earnings Calls
Group project testing whether sentiment and topics in earnings call transcripts could predict stock price changes. I cleaned the web-scraped transcripts and ran the sentiment analysis and topic extraction.

*Tools*: Python, NLTK

<a href="https://github.com/dyuthiii/CorpEarningCallNLP" target="_blank">Code</a>

### Topic Modeling Taylor Swift's Albums
Used Latent Dirichlet Allocation on lyrics to test whether all of Taylor Swift's songs really do sound the same. (They don't.)

*Tools*: Python, gensim, spaCy, NLTK

<a href="https://github.com/dyuthiii/Taylor-Swift-Albums-LDA" target="_blank">Code</a> | <a href="{{ base_path }}/posts/2025/08/2025-08-01-blog-post-3/">Blog post</a>
