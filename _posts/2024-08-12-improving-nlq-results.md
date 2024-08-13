---
author: pedwin-scottlogic
title: Improving natural language query results
summary: |
  Demonstrating how the results of using AI tools to generate SQL from natural
  language queries can be improved by adding table descriptions and Golden SQL,
  and the need for test cases to avoid regressions when making changes.
category: Artificial Intelligence
tags:
  - AI
  - SQL
  - Natural Language Query
---
# Improving natural language query results

Getting useful information from databases can be difficult if you're not
familiar with SQL (Structured Query Language). With the increase in popularity
of AI, there are now several tools available which can use AI to take a natural
language query (e.g. "What is the capital of Outer Mongolia?"), some existing
knowledge of a database populated with relevant data, and produce a SQL
statement (e.g. `SELECT city FROM worldcities WHERE country = 'Mongolia' AND
capital = 'primary'`) that will hopefully give a useful answer.

This can rapidly become more complex as the question becomes only slightly more
complicated. For example, "What is the population of the capital city of
Mongolia?" might require combining information from multiple tables, perhaps
necessitating the use of a `JOIN` statement. In these cases, the tool must be
given more information about the database schema and the relationships between
various tables in order to produce SQL that gives the right answer. In many
cases, there are multiple ways to write SQL that will give the correct answer to
the question, and the "best" SQL may be quite subjective based on trade-offs
between performance and readability.

Apart from having knowledge of the database schema, the tool can also be given
some assistance in improving the quality of its results. Two common ways of
doing this are by providing descriptions of each table and its columns, or by
the use of "Golden SQLs": natural language questions combined with the SQL that
would provide the expected answer.

Let's take a look at some examples of questions, the SQL that we might obtain
without giving the tool any extra information other than the database schema,
and how we might improve the quality of the results. For these examples, we'll
be using data from a climate change database which we've been working with as
part of a tool to help people ask questions about climate change data (see
[climate change data blog post](https://blog.scottlogic.com/category/ai.html)).
We'll be restricting ourselves to fairly simple questions about populations, but
this also requires knowledge of regions (e.g. countries and continents). The
tool we're using for generating these queries is
[Dataherald](https://www.dataherald.com/).

We'll start by doing the minimum configuration necessary: point Dataherald at
the database, and tell it to scan the schema to discover the tables and columns
that exist. We haven't yet given it any more context about how these tables
relate to each other, but it might already be able to infer some information
from the existing data. As we start prompting it, it will acquire knowledge
about the relationships between these tables and the data that each contains.

Depending on the results we obtain for a set of example questions, we can
attempt to address any issues by first providing basic descriptions of the
tables and columns (including foreign key references), then expanding on those
descriptions. Finally, we'll add a single Golden SQL to illustrate our expected
answer for question.

Here are the questions we'll be using to evaluate the quality of the answers
provided by Dataherald:

1. "Display population growth in Britain"
1. "Show me a line chart with the global population from 1980 onwards"
1. "How many centarians were there in the world in 2020?" (Yes, the misspelling
   is deliberate.)
1. "Compare US population vs China population 1970-2010"

> 📝 N.B. As is common with AI tools, the answers provided might change with
> each run, so the examples shown here might be different each time.

### Round 1: Schema scan only

#### Question 1: "Display population growth in Britain" <font color='green'>✔</font>

{% highlight sql %}
-- Display population growth in the United Kingdom
SELECT year, total
FROM population_data
WHERE region_id = 75
ORDER BY year;
{% endhighlight %}

|year|total|
|----|-------:|
|1950|50055068|
|1951|50271908|
|1952|50484812|
|…|…|

This is a good start. Dataherald has already inspected the database to find the
appropriate value for `region_id`, using United Kingdom as an effective synonym
for Britain, and has used this to filter the `population_data` table to only
show the appropriate data. Not necessarily the most efficient way of doing it,
but it's correct.

#### Question 2: "Show me a line chart with the global population from 1980 onwards" <font color='red'>✘</font>

{% highlight sql %}
-- Select the year and total population from the population_data table
-- Filter the results to include only the years from 1980 onwards
SELECT year, total
FROM public.population_data
WHERE year >= 1980
ORDER BY year;
{% endhighlight %}

|year|total|
|----|----:|
|1980|5736092|
|1980|3147166|
|1980|4408239|
|…|…|

Not so good. This query is giving us total population for each individual
region and year from 1980 onwards, not just the total **global** population.

#### Question 3: "How many centarians were there in the world in 2020?" <font color='red'>✘</font>

{% highlight sql %}
-- SQL query to find the number of centenarians in the world in 2020
SELECT SUM(age_over_100) AS centenarians_2020
FROM population_data
WHERE year = 2020;
{% endhighlight %}

|centenarians_2020|
|----------------:|
|3088375|

Over three million? That seems like a lot. Again, this isn't taking account of
the `population_data` table containing multiple regions, so the `SUM()` will be
counting centenarians in each country, plus counting them again as members of a
group of countries, continents, world… still, at least it's coped with our
deliberate misspelling of 'centenarians'.

#### Question 4: "Compare US population vs China population 1970-2010" <font color='green'>✔</font>

{% highlight sql %}
-- Compare the population of the US and China from 1970 to 2010
SELECT
    year,
    SUM(CASE WHEN region_id = 39 THEN total ELSE 0 END) AS china_population,
    SUM(CASE WHEN region_id = 224 THEN total ELSE 0 END) AS us_population
FROM
    population_data
WHERE
    year BETWEEN 1970 AND 2010
GROUP BY
    year
ORDER BY
    year;
{% endhighlight %}

|year|china_population|us_population|
|----|----------------|-------------|
|1970|822534460.0     |200328340.0  |
|1971|843285440.0     |202907920.0  |
|1972|862840400.0     |205238380.0  |
|…|…|…|

This might not be the most efficient way of writing this query, but the answer
is correct. It's using existing knowledge of `region` table to sum the totals by
year for each of the requested countries.

**Score: 2/4** Needs improvement

### Round 2: Add table and column descriptions

Let's see if we get better answers by giving Dataherald some basic descriptions
of the tables and columns.

#### Question 1: "Display population growth in Britain" <font color='green'>✔</font>

{% highlight sql %}
-- Display population growth in Britain (United Kingdom)
SELECT year, total
FROM public.population_data
WHERE region_id = 75
ORDER BY year;
{% endhighlight %}

|year|total|
|----|----:|
|1950|50055068|
|1951|50271908|
|1952|50484812|
|…|…|

The same query (and results) as before, but with schema information included. We've also been
provided with a better comment explaining the use of United Kingdom instead of
Britain.

#### Question 2: "Show me a line chart with the global population from 1980 onwards" <font color='red'>✘</font>

{% highlight sql %}
-- Select the year and the sum of the total population for each year from 1980 onwards
SELECT year, SUM(total) AS global_population
FROM population_data
WHERE year >= 1980
GROUP BY year
ORDER BY year;
{% endhighlight %}

|year|global_population|
|----|----------------:|
|1980|28185725937.0|
|1981|28724967444.0|
|1982|29283823541.0|
|…|…|

It's slightly better, as we're only getting one row per year, but we're getting
the total population of all defined regions, not just the global population.

#### Question 3: "How many centarians were there in the world in 2020?" <font color='red'>✘</font>

{% highlight sql %}
-- SQL query to find the number of centenarians in the world in 2020
SELECT SUM(age_over_100) AS centenarians_2020
FROM public.population_data
WHERE year = 2020;
{% endhighlight %}

|centenarians_2020|
|----------------:|
|3088375|

Apart from the inclusion of the schema name, this is exactly the same as before.
We still need to provide some more information about this kind of question to
get an accurate answer.

#### Question 4: "Compare US population vs China population 1970-2010" <font color='green'>✔</font>

{% highlight sql %}
-- Compare the population of the US and China from 1970 to 2010
SELECT
    year,
    SUM(CASE WHEN region_id = 39 THEN total ELSE 0 END) AS china_population,
    SUM(CASE WHEN region_id = 224 THEN total ELSE 0 END) AS us_population
FROM
    population_data
WHERE
    year BETWEEN 1970 AND 2010
GROUP BY
    year
ORDER BY
    year;
{% endhighlight %}

|year|china_population|us_population|
|----|---------------:|------------:|
|1970|822534460.0|200328340.0|
|1971|843285440.0|202907920.0|
|1972|862840400.0|205238380.0|
|…|…|…|

Interestingly, despite the perceived complexity, this is exactly the same query
as we got before. The extra information we provided doesn't appear to have
affected the resulting query.

**Score: 2/4** Still needs work

### Round 3: Improved the description of `region_id` column

Previously we'd only told Dataherald that the `region_id` column was a foreign
key to the `region` table, but we hadn't give it any further information. Let's
add this description and see if it helps:

```"description": "The ID of the region which this information relates to. This relates to entries in the region table, and may be an individual country, a group of countries, north or south hemisphere, or the entire world."```

### Question 1: "Display population growth in Britain" <font color='green'>✔</font>

{% highlight sql %}
-- Retrieve the population growth in the United Kingdom
SELECT year, total
FROM public.population_data
WHERE region_id = 75
ORDER BY year;
{% endhighlight %}

|year|total|
|----|----:|
|1950|50055068|
|1951|50271908|
|1952|50484812|
|…|…|

Despite the massive hint we've given about the `region_id` column, we're still
getting exactly the same query as before. This isn't necessarily a problem, but
for more complicated questions or more generic answers, we'd probably prefer
that the query used a `JOIN` rather than looking up the `region_id` separately.
We could probably address this using another Golden SQL.

### Question 2: "Show me a line chart with the global population from 1980 onwards" <font color='green'>✔</font>

{% highlight sql %}
-- Retrieve global population data from 1980 onwards for the region_id corresponding to "World"
SELECT year, total
FROM public.population_data
WHERE region_id = 164 AND year >= 1980
ORDER BY year;
{% endhighlight %}

|year|total|
|----|----:|
|1980|4444008000|
|1981|4524627500|
|1982|4607984600|
|…|…|

That's more like it. The description of the `region_id` column seems to have
helped Dataherald to understand that the table contains data for multiple
regions, and it understands that the global population is represented by the
'World' region.

### Question 3: "How many centarians were there in the world in 2020?" <font color='red'>✘</font>

{% highlight sql %}
-- SQL query to find the number of centenarians in the world in 2020
SELECT SUM(age_over_100) AS centenarians_2020
FROM public.population_data
WHERE year = 2020;
{% endhighlight %}

|centenarians_2020|
|----------------:|
|3088375|

Despite getting the answer to Question 2 correct now, we're still getting the
same incorrect answer here. Hopefully a Golden SQL will help with this, if we
give an appropriate example.

### Question 4: "Compare US population vs China population 1970-2010" <font color='green'>✔</font>

{% highlight sql %}
-- Compare the population of the US and China from 1970 to 2010
SELECT
    year,
    SUM(CASE WHEN region_id = 39 THEN total ELSE 0 END) AS china_population, -- Population of China
    SUM(CASE WHEN region_id = 224 THEN total ELSE 0 END) AS us_population -- Population of the United States
FROM
    public.population_data
WHERE
    year BETWEEN 1970 AND 2010 -- Filter for the years 1970 to 2010
GROUP BY
    year -- Group by year to get annual population data
ORDER BY
    year; -- Order the results by year
{% endhighlight %}

|year|china_population|us_population|
|----|---------------:|------------:|
|1970|822534460.0|200328340.0|
|1971|843285440.0|202907920.0|
|1972|862840400.0|205238380.0|

This is still using the predetermined values of `region_id`, just with a little
more commenting about what they represent.

**Score: 3/4** Much better

### Round 4: Added a single Golden SQL

Let's see if we can address our remaining incorrect answer by providing
Dataherald with more context about how to obtain certain results and the
relationship between tables, by means of a Golden SQL statement.

> Question:
>
> *What percentage of the global population was under 15 in each year
> of the 1970s?*
>
> Answer:
>
> ```select year, (pd.age_under_15::decimal / pd.total) * 100 as percent_under_15 from public.population_data pd inner join public.region r on pd.region_id = r.id and name = 'World' where year between 1970 and 1979;```

#### Question 1: "Display population growth in Britain" <font color='green'>✔</font>

{% highlight sql %}
-- Select the year and total population for the United Kingdom to display population growth
SELECT
    pd.year,
    pd.total
FROM
    public.population_data pd
INNER JOIN
    public.region r
ON
    pd.region_id = r.id
WHERE
    r.name = 'United Kingdom'
ORDER BY
    pd.year;
{% endhighlight %}

|year|total|
|----|----:|
|1950|50055068|
|1951|50271908|
|1952|50484812|
|…|…|

Excellent. We're still getting a valid answer, but now it's using a `JOIN` to
find the appropriate `region_id` rather than looking it up separately. This is
likely to be a more efficient method for more complicated scenarios.

#### Question 2: "Show me a line chart with the global population from 1980 onwards" <font color='green'>✔</font>

{% highlight sql %}
-- Select the year and total population from 1980 onwards for the global population
SELECT
    pd.year,
    pd.total
FROM
    public.population_data pd
INNER JOIN
    public.region r
ON
    pd.region_id = r.id
WHERE
    r.name = 'World'
    AND pd.year >= 1980
ORDER BY
    pd.year;
{% endhighlight %}

|year|total|
|----|----:|
|1980|4444008000 |
|1981|4524627500 |
|1982|4607984600 |
|…|…|

Another good improvement. We're still getting a correct answer, but this has
also been changed to utilise a `JOIN` to find the appropriate `region_id` for
the 'World' region.

#### Question 3: "How many centarians were there in the world in 2020?" <font color='green'>✔</font>

{% highlight sql %}
-- Select the number of centenarians in the world in 2020
SELECT pd.age_over_100
FROM public.population_data pd
INNER JOIN public.region r ON pd.region_id = r.id
WHERE r.name = 'World' AND pd.year = 2020;
{% endhighlight %}

|age_over_100|
|-----------:|
|547543|

Success! The Golden SQL we provided appears to have helped Dataherald comprehend
how to filter the results here, so we only get the centenarians for the 'World'
region in 2020.

#### Question 4: "Compare US population vs China population 1970-2010" <font color='red'>✘</font>

{% highlight sql %}
-- SQL query to compare the populations of the US and China from 1970 to 2010
SELECT
    r.name AS country,
    pd.year,
    pd.total AS population
FROM
    public.population_data pd
INNER JOIN
    public.region r
ON
    pd.region_id = r.id
WHERE
    r.name IN ('United States', 'China')
    AND pd.year BETWEEN 1970 AND 2010
ORDER BY
    pd.year, r.name;
{% endhighlight %}

|country|year|population|
|-------|----|---------:|
|China  |1970|822534460|
|United States|1970|200328340|
|China  |1971|843285440|
|United States|1971|202907920|
|China  |1972|862840400|
|United States|1972|205238380|
|…|…|…|

Disaster! We've broken this. Dataherald now gives a different result, generating
lines of data for each of the two countries per year, rather than a single line
per year. This could probably be addressed by adding further SQL examples to
illustrate the expected output.

**Score: 3/4 (again)** Disappointing regression

### Cost

Note that the additional information we're providing to Dataherald in the form
of descriptions and Golden SQL is not without cost, above the obvious one of the
time taken to for us to consider the existing answers and craft appropriate
information. Here's the token counts for each round of questions:

| Round | Question 1 | Question 2 | Question 3 | Question 4 | Total |
|:-----:|-----------:|-----------:|-----------:|-----------:|------:|
| 1 | 17574 | 11586 | 5823 | 18755 | **53738** |
| 2 | 26176 | 10620 | 6987 | 33238 | **77021** |
| 3 | 22333 | 18846 | 9569 | 27180 | **77928** |
| 4 | 37996 | 13796 | 16262 | 28371 | **96425** |

We can see that the total token count has almost doubled as we've added
descriptions and Golden SQLs, and we're still not getting completely accurate
results. There's likely some internal caching going on as well, meaning that
information determined in previous rounds of questions is being used to enhance
future answers; running from a clean slate each time would likely result in
bigger increases in token usage. We need to be careful not to just keep adding
more and more information in the hope of getting better results, as this could
cause the costs to escalate quickly.

### Conclusion

Even when using existing tools such as Dataherald that help with the process,
generating SQL from natural language queries is not straightforward. It's
important to ensure that you test this tooling just as you would for any other
piece of code, by establishing a set of test cases that you can examine each
time you make changes to the system, prior to releasing those changes to
production. The same principles as you'd apply to any other piece of production
code need to be applied here to ensure that you don't unwittingly introduce
regressions when optimising for specific cases.

A further complication is that the answers provided by these tools are not
deterministic. There are often multiple ways of providing data which will
potentially answer a natural language question, and different ways of writing
the SQL that could generate any of those answers, so we can't just compare the
SQL to what we expect, or even the data extracted by running that SQL.

Tools such as [promptfoo](https://www.promptfoo.com/) can be a great help for
cases like this, as they're flexible enough to allow you to connect to a variety
of AI systems and provide test cases where the answers can be evaluated to
ensure they are sensible.

Scott Logic have a wealth of experience in prompt engineering and testing
practices to help you get the most out of your AI tools and investment. If
you'd like to know more about how we can help you, please get in touch.
