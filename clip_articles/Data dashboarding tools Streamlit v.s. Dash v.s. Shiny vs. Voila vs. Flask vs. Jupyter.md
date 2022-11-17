# Data dashboarding tools | Streamlit v.s. Dash v.s. Shiny vs. Voila vs. Flask vs. Jupyter
[Data dashboarding tools | Streamlit v.s. Dash v.s. Shiny vs. Voila vs. Flask vs. Jupyter](https://www.datarevenue.com/en-blog/data-dashboarding-streamlit-vs-dash-vs-shiny-vs-voila) 

 ![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-37-31/2c512d60-14b5-4455-beb7-d910538b8b6f.png?raw=true)

Over the last three years, Dash and Streamlit have surged in popularity as all-in-one dashboarding solutions.

## **Data dashboards – Tooling and libraries**

Nearly every company is sitting on valuable data that internal teams need to access and analyze. Non-technical teams often request tooling to make this easier. Instead of having to poke a data scientist for every request, these teams want dynamic dashboards where they can easily run queries and see custom, interactive visualizations.

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-37-31/f7b0de0b-96b3-4107-a7c5-0fd74fcbed00.png?raw=true)

Data dashboards can make data more accessible to your non-technical teams

A data dashboard consists of many different components. It needs to:

-   **Analyze:** Manipulate and summarize data using a backend library such as Pandas.
-   **Visualize:** Create plots and graphs of the data using a graphing library such as Bokeh.
-   **Interact:** Accept user input using a frontend library such as React.
-   **Serve:** Listen for user requests and return webpages using a web server such as Flask.

In the past, you’d have had to waste a significant amount of time writing all the “glue” code to join these components together. But with newer libraries like Streamlit and Dash, these components come in a single package.

Still, figuring out which library to use can be challenging. Here’s how they compare as well as some guidance on how to choose which one is best for your project.

Do you want more detailed tooling comparisons that cut through the marketing-speak?

Sign up to our weekly newsletter.

Thank you! Your submission has been received!

Oops! Something went wrong while submitting the form.

## **Just tell me which one to use**

As always, “it depends” – but if you’re looking for a quick answer, you should probably use:

-   **Dash** if you already use Python for your analytics and you want to build production-ready data dashboards for a larger company.
-   **Streamlit** if you already use Python for your analytics and you want to get a prototype of your dashboard up and running as quickly as possible.
-   **Shiny** if you already use R for your analytics and you want to make the results more accessible to non-technical teams.
-   **Jupyter** if your team is very technical and doesn’t mind installing and running developer tools to view analytics.
-   **Voila** if you already have Jupyter Notebooks and you want to make them accessible to non-technical teams.
-   **Flask** if you want to build your own solution from the ground up.
-   **Panel** if you already have Jupyter Notebooks, and Voila is not flexible enough for your needs.

## **Quick overview**

Not all the libraries are directly comparable. For example, Dash is built on top of Flask, and Flask is a more general framework for web application development. Similarly, each library focuses on a slightly different area.

-   **Streamlit, Dash**, and **Panel** are full dashboarding solutions, focused on Python-based data analytics and running on the **Tornado** and **Flask** web frameworks.
-   **Shiny** is a full dashboarding solution focused on data analytics with R.
-   **Jupyter** is a notebook that data scientists use to analyze and manipulate data. You can also use it to visualize data.
-   **Voila** is a library that turns individual Jupyter notebooks into interactive web pages.
-   **Flask** is a Python web framework for building websites and apps – not necessarily with a data science focus.

Some of these libraries have been around for a while, and some are brand new. Some are more rigid, and have their own structure, while others are flexible and can adapt to yours. Some focus on specific languages. Here’s a table showing the tradeoffs:

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2022-11-17%2019-37-31/7d82d420-ccc4-4c27-8f83-5c99a2fb4f87.png?raw=true)

We’ve compared these libraries on:

-   **Maturity:** Based on the age of the project and how stable it is.
-   **Popularity:** Based on adoption and GitHub stars.
-   **Simplicity:** Based on how easy it is to get started using the library.
-   **Adaptability:** Based on how flexible and opinionated the library is.
-   **Focus:** Based on what problem the library solves.
-   **Language support:** The main languages the library supports.

These are not rigorous or scientific benchmarks, but they’re intended to give you a quick overview of how the tools overlap and how they differ from each other. For more details, see the head-to-head comparison below.

## **Streamlit vs. Dash**

Streamlit and Dash are the two most similar libraries in this set. They are both full dashboarding solutions built with Python, and both include components for data analysis, visualization, user interaction, and serving. 

Although they’re both open source, Dash is more focused on the enterprise market and doesn’t include all the features (such as job queues) in the open source version. By contrast, Streamlit is fully open source. 

Streamlit is more structured and focused more on simplicity. It only supports Python-based data analysis and has a limited set of widgets (for example, sliders) to choose from.

Dash is more adaptable. Although it’s built with Python and pushes users towards its own plotting library (Plotly), it’s also compatible with other plotting libraries and even other languages, such as R or Julia. 

-   **Use** **Streamlit** if you want to get going as quickly possible and don’t have strong opinions or many custom requirements.
-   **Use Dash** if you need something more flexible and mature, and you don’t mind spending the extra engineering time. 

## **Streamlit vs. Shiny**

Streamlit is a dashboard tool based on Python, while Shiny uses R. Both tools focus on turning data analysis scripts into full, interactive web applications. 

Because Python is a general-purpose language while R is focused solely on data analytics, the web applications you build with Streamlit (based on the Tornado web server) are more powerful and easier to scale to production environments than those built with Shiny. 

Shiny integrates well with plotting libraries in the R ecosystem, such as ggplot2, while Streamlit integrates with Python plotting libraries such as Bokeh or Altair.

-   **Use** **Shiny** if you prefer doing data analysis in R and have already invested in the R ecosystem.
-   **Otherwise use Streamlit** (or Dash – see above).

## **Streamlit vs. Voila**

Streamlit is a complete data dashboarding solution, while Voila is a simpler and more limited tool that lets you convert existing Jupyter Notebooks into basic data dashboards and serve them as web applications to non-technical users.

Like Streamlit, Voila is built on top of the Tornado web framework, so you can use Jupyter notebooks along with Voila to get something broadly similar to Streamlit. But Streamlit is more flexible (it doesn’t require you to use Jupyter), while Voila can be simpler (provided you already have Jupyter Notebooks you want to present).

Voila uses Jupyter’s widget library, while Streamlit uses custom widgets – so if you’re already familiar with Jupyter, you’ll find Voila easier to work with.

-   **Use Streamlit** If you’re looking for an all-in-one solution.
-   **Use Voila** if you already have Jupyter Notebooks and are looking for a way to serve them.

## **Streamlit vs. Panel**

**Streamlit** and **Panel** are both data dashboarding solutions, but **Panel** - like Voila - integrates better with Jupyter Notebooks.

Engineers can use either Streamlit or Panel to develop interactive data dashboards for non-technical users, but they might use Panel for internal data exploration as well.

-   **Use** **Streamlit** if you are looking for a more mature data dashboarding solution and your primary goal is to develop dashboards for non-technical people.
-   **Use Panel** if you already use Jupyter Notebooks and need something more powerful than **Voila** to turn them into dashboards.  

## **Streamlit vs. Jupyter Notebooks**

Streamlit is a full data dashboarding solution, while Jupyter Notebooks are primarily useful to engineers who want to develop software and visualizations. Engineers use Streamlit to build dashboards for non-technical users, and they use Jupyter Notebooks to develop code and share it with other engineers.

Combined with add-ons such as Voila, Jupyter Notebooks can be used similarly to Streamlit, but data dashboarding is not their core goal.

-   **Use Streamlit** if you need dashboards that non-technical people can use.
-   ‍**Jupyter Notebooks** are best if your team is mainly technical and you care more about functionality than aesthetics.

## **Streamlit vs. Flask**

Streamlit is a data dashboarding tool, while Flask is a web framework. Serving pages to users is an important but small component of data dashboards. Flask doesn’t have any data visualization, manipulation, or analytical capabilities (though since it’s a general Python library, it can work well with other libraries that perform these tasks). Streamlit is an all-in-one tool that encompases web serving as well as data analysis.

-   **Use Streamlit** if you want a structured data dashboard with many of the components you’ll need already included. Use Streamlit  if you want to build a data dashboard with common components and don’t want to reinvent the wheel.
-   **Use Flask** if you want to build a highly customized solution from the ground up and you have the engineering capacity.

## **Dash vs Panel**

**Dash** and **Panel** are both data dashboarding solutions, but Panel integrates better with Jupyter Notebooks. While Dash lets developers customize the front-end by writing HTML and CSS, Panel automatically creates front-ends based only on Python syntax.

Engineers can use either to build interactive dashboards for non-technical users, but they might use Panel for data exploration too.

-   **Use** **Dash** if you want to build more customized data dashboards for non-technical users.
-   **Use** **Panel** if you already use Jupyter Notebooks and want more flexibility than is offered by Voila.

## **Dash vs. Shiny**

Dash and Shiny are both complete data dashboarding tools, but Dash lives mainly in the Python ecosystem, while Shiny is exclusive to R. 

Dash has more features than Shiny, especially in its enterprise version, and it's more flexible. Python is a general-purpose programming language, while R is focused solely on data analytics. Some data scientists prefer R for its mature libraries and (often) more concise code. Engineers usually prefer Python, since it conforms more closely to other languages.

-   **Use Dash** if your team prefers Python.
-   **Use Shiny** if your team prefers R.

## **Dash vs. Voila and Jupyter Notebooks**

Dash is an all-in-one dashboarding solution, while Voila can be combined with Jupyter Notebooks to get similar results. Dash is more powerful and flexible, and it’s built specifically for creating data dashboards, while Voila is a thin layer built on top of Jupyter Notebooks to convert them into stand-alone web applications.

-   **Use Dash** if you want to build a scalable, flexible data dashboarding tool.
-   **Use Voila** if you have existing Jupyter Notebooks you want your non-technical teams to be able to use.

## **Dash vs. Flask**

Dash is built on top of Flask and uses Flask as its web routing component, so it’s not very meaningful to compare them head-to-head. Dash is a data dashboarding tool, while Flask is a minimalist, generic web framework. Flask has no data analytics tools included, although it can work with other Python libraries that do analytics.

-   **Use Dash** if you want to build a data dashboard.
-   **Use** **Flask** if you want to build a far more generic web application and to choose every component in it.

## **Shiny vs. Voila + Jupyter Notebooks**

Shiny is a data dashboarding solution for R. While you can use Voila and Jupyter Notebooks with R, these are tools that focus primarily on the Python ecosystem.

-   **Use Shiny** if you already do your data analytics in R.
-   **Use Voila** if you already have Jupyter Notebooks you want to make more accessible.

## **Shiny vs. Flask**

Shiny is a data dashboarding tool built in R. Flask is a web framework built in Python. Shiny works well with R plotting libraries, such as ggplot2. Flask doesn’t have any data analysis tools built in by default.

-   **Use Shiny** if you’re building a data dashboard and you want to do your data analysis with R.
-   **Use Flask** if you want to build a generic web application from the ground up.

## **Voila vs. Flask**

Voila is a library to convert Jupyter Notebooks to stand-alone web applications and serve them using Tornado. Like Tornado, Flask is a generic web framework. While it would be possible to use Flask to serve Jupyter Notebooks, you would have to reimplement most of the Voila library – so unless you have a very specific reason, it’s better to simply use Voila.

## **Final remarks**

All the tools we’ve covered here can help you access the value locked away in your existing data. One common mistake we see teams make is getting too tied up in choosing which tools to use, rather than focusing on the data itself. While using the wrong tools can definitely hinder your analysis, it’s more common for teams to get bogged down by so-called [Bikeshedding](https://exceptionnotfound.net/bikeshedding-the-daily-software-anti-pattern/): spending too much time debating details that aren’t very important.

If you’d like to chat about exploring your data and turning it into more revenue, [book a free call with our CEO](https://datarevenue.com/en-contact).
