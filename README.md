# Data512BiasInData

### Introduction

Among the numerous articles hosted on Wikipedia and curated by its users are various articles covering the lives and careers of notable politicians from around the world. For many data-focused political research projects, Wikipedia's information serves as an easily accessible and widely repected data source. However, not all Wikipedia articles are created equal.

As part of its community curation systems, Wikipedia allows for every article to be assigned a quality rating indicated how thorough, well-reseached, and transparent it content and sources are. Due to the fact that this process requires significant human effort, not all articles have such a grade assigned to them. However, Wikipedia's ORES API allows for quality rating to be assigned through a machine learning model trained on the prior quality ratings given by Wikipedia's community.

Through the use of this system, it becomes reasonable to ask the questions that this analysis will seek to answer: Is there a bias in quality of Wikipedia politician articles across different countries and regions? And if so, what patterns describe the nature of of this bias?

### Data Sources and Licencing

In this analysis, all page data will be obtained through either the Wikipedia Analytics Page Info API, an API that provides access to Wikipedia's page info according to its terms of use, or the Wikipedia ORES API [(documentated by wikimedia)](https://www.mediawiki.org/wiki/API:Info), which uses machine learning to assign predicted article quality scores to articles that do not already have a community score assigned to them. By the Wikimedia Foundation's terms of use, site and API users are free to read, print, share, and reuse the contents of their sites under the condition that the content in question is used responsibly, with civility, lawfully, without intent to harm the Foundation or other human beings, with full disclosure, and while being held fully responsible for the consequences of how this content is used or interpretted. As the data from this API will be used for a strictly educational exploration of copyright free material with no intent to monetize, this notebook's dataset falls under the Foundation's terms of use.

In order to obtain this API data, two other datasets were obtained. Firstly, [the Wikipedia category of politicians by nationality](https://en.wikipedia.org/wiki/Category:Politicians_by_nationality) was crawled to obtain a list of articles on political figures across the world. This list was obtained for the express purpose of querying the Page Info API to obtain page info for articles on politicians. Similarly, the [world population datasheet](https://www.prb.org/international/indicator/population/table/) was downloaded from the Population Reference Bureau in order to assign populations to each country and calculate per-capita figures for article quantity and quality.

In addition to this, two API templates provided by David McDonald's [sample wikimedia](wp_page_info_example.ipynb) [notebooks](wp_ores_liftwing_example.ipynb) were used to help perform API calls to the Page Info API and the ORES API. This code was used under its CC-BY licence, with no substantial alterations whatsoever. Both of the API call constants and API call functions utilized in the analysis notebook are the work of David McDonald.

### Data Collection

To begin with, the list of Wikipedia politician articles was cleaned of articles containing the strings `"Presid"`, `"Ministry"`, and `"Ministers"`. Though the data crawler that obtained this was _intended_ to obtain only the titles of articles about specific politicians, several articles about various political _positions_ were erroneously included in the dataset instead. Filtering out the above strings removes several of these incorrect articles while leaving all articles pertaining to specific people in the dataset.

Curiously, a secondary glance at the list reveals a much more significant issue to the analysis. Several political figures have separate articles for their policial histories and their overall lives. In particular, politicians from impoverished, obscure, or otherwise poorly-documented countries seem to be much more likely to have their articles split in this way. Less than 20 such articles appear to exist in the dataset, so it is unlikely to influence the final analysis, but this trend is worth noting for any attempting to build off of the datasets produced in this work.

Using the articles in the now cleaned list, the Wikimedia Page Info API was called to obtain the page info for every article of interest to the analysis. Within this page info is the "revision ID" of the most recent version of the article. This data will be used to call the ORES API, as the API requires a apecific revision of the article to be provided in each API request.

Unfortunately, several of the page info responses returned data with missing article IDs and revision IDs. This typically happens because an article lacks an english version in its current state. Regardless of the cause, the 8 articles containing this issue cannot be used to call the ORES API and so must be dropped. As they comprised a miniscule portion of the dataset, this is not anticipated to introduce any inaccuracies to the analysis.

Following this, the Population Reference Bureau's world population datasheet was read in for future merging.

With the revision IDs obtained from the Page Info API calls, the ORES WingLift API could then be called to obtain the quality score estimates for each article. Due to the long nature of the the ORES API call, some articles run into request errors and return an http error instead of the ORES API prediction. During the initial run of this analysis, the articles for 'Yabre Juliette Kongo', 'Vincent de Paul Ahanda', and 'Mohammed El Senussi' had to be dropped from the dataset due to their API calls returning an errored response.

With all three datasets obtained, the were then cleaned and merged in preparation for the final analysis.

As part of this:

Any articles that lack a quality estimate were removed as they cannot be used for this analysis. Approximately ten articles were dropped in this way.

Following this, the quality and page info dataset were be neatly joined on their shared article name columns. And to simplify later analysis of the assigned quality scores, a column was be added to the merged dataset containing the quality category that has the highest probability assigned to it by the ORES ML model.

With the page info and ORES data combined, the population and region data was added next. To accomplish this, each article's country was matched to the region associated with it in the population file's hierarchy. This can be done due to the simple fact that each region in the population dataset is written in all caps and comes immediately before the names of the countries associated with it. Thus, the region was obtained by finding the country name in the dataset and working back up the list until a region name is found.

Then, the population for that country was obtained by finding the matching row in the dataset.

During this process, articles from three different countries were found to have no corresponding region in the population dataset. As this renders further analysis impossible, these countries were dropped from the dataset and their names were stored as a simple list of strings in `"wp_countries-no_match.txt"`.

With the three datasets combined, only a small number of final adjustments were performed. In order to simplify future analysis, a boolean column containing the value of whether or not a given article was classified as 'high quality' (`FA` or `GA` in the ORES API) was added to the dataset, the columns were renamed for clarity, and the dataset was saved as `"wp_politicians_by_country.csv"` for use in future work.

The format of the final `"wp_politicians_by_country.csv"` dataset is:

```
{
    'article_title': <The title of the Wikipedia article>,
    'url': <The URL of the article>,
    'country': <The country associated with the politician in the article>,
    'B': <The probability that the article is of "B" quality according to the ORES ML model>,
    'C': <The probability that the article is of "C" quality according to the ORES ML model>,
    'FA': <The probability that the article is of "FA" quality according to the ORES ML model>,
    'GA': <The probability that the article is of "GA" quality according to the ORES ML model>,
    'Start': <The probability that the article is of "Start" quality according to the ORES ML model>,
    'Stub': <The probability that the article is of "Stub" quality according to the ORES ML model>,
    'revision_id': <The ID of the most recent revision of this article>,
    'article_quality': <The article quality with the highest predicted probability according to the ORES ML model>,
    'region': <The region associated with this politician's country>,
    'population': <The population of this politician's country>,
    'high_quality': <True if article_quality is "FA" or "GA". False otherwise.>
}
```

### Analysis and Results

Using the final combined dataset, the six main questions of this analysis can be explored.

The first two of these questions are:
* What 10 countries have the most politician articles per capita?

and

* What 10 countries have the _fewest_ politician articles per capita?

To answer these first two questions, the datset was grouped by country and population so that the count of articles within each country could be calculated and divided by each country's population to obtain the number of articles per million people. Resulting in the following table:

|     | country                        |   population |   num_articles |   articles_per_capita |
|----:|:-------------------------------|-------------:|---------------:|----------------------:|
|   4 | Antigua and Barbuda            |          0.1 |             33 |              330      |
|  51 | Federated States of Micronesia |          0.1 |             14 |              140      |
|  93 | Marshall Islands               |          0.1 |             13 |              130      |
| 149 | Tonga                          |          0.1 |             10 |              100      |
|  12 | Barbados                       |          0.3 |             25 |               83.3333 |
|  98 | Montenegro                     |          0.6 |             38 |               63.3333 |
| 125 | Seychelles                     |          0.1 |              6 |               60      |
|  90 | Maldives                       |          0.6 |             33 |               55      |
|  17 | Bhutan                         |          0.8 |             44 |               55      |
| 121 | Samoa                          |          0.2 |              8 |               40      |

Looking at the above table, it can be seen that nearly all countries with a high per-capita article count achieve this due to their low populaations. Regardless of how small these countries are, their politicians possess a baseline level of importance that ensures that they will have articles written about them on Wikipedia.

Sorting this dataset in ascending order instead, we obtain:


|     | country       |   population |   num_articles |   articles_per_capita |
|----:|:--------------|-------------:|---------------:|----------------------:|
|  66 | India         |       1428.6 |            151 |              0.105698 |
| 122 | Saudi Arabia  |         36.9 |              5 |              0.135501 |
| 164 | Zambia        |         20.2 |              3 |              0.148515 |
| 108 | Norway        |          5.5 |              1 |              0.181818 |
|  70 | Israel        |          9.8 |              2 |              0.204082 |
|  45 | Egypt         |        105.2 |             32 |              0.304183 |
|  37 | Cote d'Ivoire |         30.9 |             10 |              0.323625 |
| 100 | Mozambique    |         33.9 |             12 |              0.353982 |
|  50 | Ethiopia      |        126.5 |             45 |              0.355731 |
| 162 | Vietnam       |         98.9 |             36 |              0.364004 |

Since the countries with a _high_ per-capita article count achieve their status through a low population count, it should come as no surprise that the countries with a _low_ per-capita article count achieve _their_ status with a high population count. Being comprised primarily of countries with some combination of sizable populations or exceptionally few articles.

The second pair of questions to be answered by this analysis are:
* What 10 countries have the most _high-quality_ politician articles per capita?

and

* What 10 countries have the fewest _high-quality_ politician articles per capita?

To answer these questions, the dataset can once again be grouped by country and population so that the total number of high quality articles for that country can be calculated and divided by the total population.

|     | country         |   population |   high_quality |   quality_per_capita |
|----:|:----------------|-------------:|---------------:|---------------------:|
|  98 | Montenegro      |          0.6 |              2 |             3.33333  |
|  86 | Luxembourg      |          0.7 |              1 |             1.42857  |
|  85 | Lithuania       |          2.9 |              4 |             1.37931  |
|  76 | Kosovo          |          1.7 |              2 |             1.17647  |
|   1 | Albania         |          2.7 |              3 |             1.11111  |
| 107 | North Macedonia |          1.8 |              1 |             0.555556 |
|  80 | Latvia          |          1.9 |              1 |             0.526316 |
| 124 | Serbia          |          6.6 |              3 |             0.454545 |
|  13 | Belarus         |          9.2 |              3 |             0.326087 |
|  95 | Moldova         |          3.4 |              1 |             0.294118 |

Much like with the results for the raw number of articles-per-capita, the countries with a high number of high-quality articles per capita uniformly have moderate or small populations. However, these countries _also_ have a relatively large number of high_quality articles in each. It may not initially appear to be the case, as each country only has a 1-4 articles each, but it turns out that exceptionally few articles in the dataset (approximately 90 in total) were assigned a high quality score.

Sorting this result dataset in ascending order instead, we can obtain the obvious answer to the inverse question:

|     | country               |   population |   high_quality |   quality_per_capita |
|----:|:----------------------|-------------:|---------------:|---------------------:|
|   0 | Afghanistan           |         42.4 |              0 |                    0 |
| 117 | Portugal              |         10.5 |              0 |                    0 |
| 114 | Paraguay              |          6.2 |              0 |                    0 |
| 113 | Papua New Guinea      |          9.5 |              0 |                    0 |
| 111 | Palestinian Territory |          5.5 |              0 |                    0 |
| 110 | Pakistan              |        240.5 |              0 |                    0 |
| 109 | Oman                  |          5   |              0 |                    0 |
| 108 | Norway                |          5.5 |              0 |                    0 |
| 105 | Niger                 |         27.2 |              0 |                    0 |
| 104 | Nicaragua             |          6.8 |              0 |                    0 |

Since very few high_quality articles exist in the dataset, countless countries have 0 high quality articles whatsoever, and thus they are all tied for the lowest per-capita count of high-quality articles about politicians.

Thus far, these analyses have been focused on comparisons across countries. However the final two questions to be answered are:

* What regions have the most politician articles per capita?

and

* What regions have the most _high-quality_ politician articles per capita?

To answer these questions, the dataset can be grouped by region and the sum of both populations and high-quality article counts can be obtained across each region while the counts of politician articles overall can be calculated separately. The article and high-quality article totals can then be divided by the population totals to obtain the per-capita values for both articles overall and high-quality articles specifically.

| region          |   article_count |   population |   articles_per_capita |
|:----------------|----------------:|-------------:|----------------------:|
| OCEANIA         |              71 |        110.8 |            0.640794   |
| NORTHERN EUROPE |             196 |       1194.5 |            0.164085   |
| CARIBBEAN       |             215 |       1369.7 |            0.156969   |
| CENTRAL AMERICA |             193 |       1477.5 |            0.130626   |
| CENTRAL ASIA    |             118 |       2203.5 |            0.0535512  |
| WESTERN ASIA    |             613 |      13469.7 |            0.0455096  |
| SOUTHERN EUROPE |             819 |      18285.8 |            0.0447889  |
| EASTERN AFRICA  |             672 |      24116.9 |            0.0278643  |
| WESTERN EUROPE  |             500 |      19064   |            0.0262274  |
| NORTHERN AFRICA |             306 |      12366.3 |            0.0247447  |
| EASTERN EUROPE  |             726 |      29582.5 |            0.0245415  |
| MIDDLE AFRICA   |             230 |       9496.8 |            0.0242187  |
| SOUTHERN AFRICA |             123 |       5954.3 |            0.0206573  |
| SOUTH AMERICA   |             564 |      33494.1 |            0.0168388  |
| SOUTHEAST ASIA  |             399 |      45336.1 |            0.00880093 |
| WESTERN AFRICA  |             513 |      58724.2 |            0.00873575 |
| EAST ASIA       |             153 |      37537.3 |            0.00407595 |
| SOUTH ASIA      |             676 |     265404   |            0.00254706 |

Once again, we can see that raw article values are highest for the major european and asian regions, but the per-capita article counts are highest in regions with small populations.

| region          |   population |   high_quality |   quality_per_capita |
|:----------------|-------------:|---------------:|---------------------:|
| NORTHERN EUROPE |       1194.5 |              7 |          0.00586019  |
| CARIBBEAN       |       1369.7 |              5 |          0.00365043  |
| SOUTHERN EUROPE |      18285.8 |             27 |          0.00147656  |
| CENTRAL AMERICA |       1477.5 |              2 |          0.00135364  |
| SOUTHERN AFRICA |       5954.3 |              4 |          0.000671783 |
| WESTERN ASIA    |      13469.7 |              7 |          0.000519685 |
| CENTRAL ASIA    |       2203.5 |              1 |          0.000453823 |
| EASTERN EUROPE  |      29582.5 |             10 |          0.000338038 |
| MIDDLE AFRICA   |       9496.8 |              3 |          0.000315896 |
| WESTERN EUROPE  |      19064   |              4 |          0.00020982  |
| NORTHERN AFRICA |      12366.3 |              2 |          0.00016173  |
| SOUTH AMERICA   |      33494.1 |              5 |          0.00014928  |
| SOUTHEAST ASIA  |      45336.1 |              5 |          0.000110287 |
| EASTERN AFRICA  |      24116.9 |              2 |          8.29294e-05 |
| WESTERN AFRICA  |      58724.2 |              3 |          5.10863e-05 |
| SOUTH ASIA      |     265404   |              3 |          1.13035e-05 |
| EAST ASIA       |      37537.3 |              0 |          0           |
| OCEANIA         |        110.8 |              0 |          0           |

From this final table, however, we can at last see a clear bias in the quality of the articles on politicians put together on Wikipedia. Compared to the other regions, the european regions have substantially more high-quality articles and a relatively small or moderate population count that makes their high-quality article count per-capita absurdly larger than the other regions.

### Reflections

Looking at the results of this analysis, several key insights stand out as being particularly worthy of note. Throughout this analysis, it could be seen that the articles from less well-known or financially influential countries were far less likely to be contructed with Wikipedia's tenets of quality. From the analyses of high-quality articles it can be seen that the bulk of high quality articles were for politicians in european regions, to a degree that cannot be explained by population alone. However, even the process of cleaning the datasets for use in assembling each table revealed the fact that politicians from less influential countries were far more likely to have their political accomplishments separated off into a small stub article separate from the article on their personal life. Perhaps unsurprisingly, Wikipedia has a noticable bias towards mainstream European politicians when it comes to the effort put into refining each article. Given that Wikipedia is curated by individuals driven by their own passion for the site, it is ultimately quite understandable that the politicians that are more influential, and thus more _visible_ in modern culture, would recieve a disproportionate amount of attention and effort on their articles.

However, this presents an obvious issue for researchers seeking to use Wikipedia as a source of unbiased political history. Of course, any attempt to use Wikipedia's information without an acknowledgement of the inherent biases of crowdsourcing information is misguided to begin with. But the thoroughness with which Wikipedia catalogues information from politicians across the work nevertheless makes it a compelling source of data for a researcher seeking to compare political careers from different regions across the world. A researcher aiming to study the differences between a typical political career path in Britain vs in the United States vs in Pakistan may make the decision to use Wikipedia as a source of comprehensive data simply due to the number of politicians it has catalogued. However, the conclusions such a researcher may draw would likely end up skewed by the fact that the information, citations, and article structure for politicians outside of europe are simply worse than they are for those specific countries.

Ultimately, any researcher attempting to use Wikipedia as a source of political career information will have to enrich the dataset to support their needs. While Wikipedia has a significant disparity in article quality, the details within ech article often come with specific details and citations for all articles greater than a stub. As such, a researcher could use Wikipedia as a baseline source to learn the basic details of a politician's life and actions, while supplementing the articles with data obtained from political science research in the country associated with each figure. As this very analysis demonstrates, Wikipedia articles can be effective sources for analysis when combined with outside datasets such as population data and when embelished by additional data from sources such as the ORES API. Despite its inconsistency across regions, it serves perfectly well as a dataset to merge other more meticulous (but less expansive) datasets onto.