How to use screen scraping tools to extract data from the web
======
![](https://opensource.com/sites/default/files/styles/image-full-size/public/lead-images/openweb-osdc-lead.png?itok=yjU4KliG)
A perfect internet would deliver data to clients in the format of their choice, whether it's CSV, XML, JSON, etc. The real internet teases at times by making data available, but usually in HTML or PDF documents—formats designed for data display rather than data interchange. Accordingly, the [screen scraping][1] of yesteryear—extracting displayed data and converting it to the requested format—is still relevant today.

Perl has outstanding tools for screen scraping, among them the `HTML::TableExtract` package described in the Scraping program below.

### Overview of the scraping program

The screen-scraping program has two main pieces, which fit together as follows:

  * The file data.html contains the data to be scraped. The data in this example, which originated in a university site under renovation, addresses the issue of whether the income associated with a college degree justifies the degree's cost. The data includes median incomes, percentiles, and other information about areas of study such as computing, engineering, and liberal arts. To run the Scraping program, the data.html file should be hosted on a web server, in my case a local Nginx server. A standalone Perl web server such as `HTTP::Server::PSGI` or `HTTP::Server::Simple` would do as well.
  * The file scrape.pl contains the Scraping program, which uses features from the `Plack/PSGI` packages, in particular a Plack web server. The Scraping program is launched from the command line (as explained below). A user enters the URL for the Plack server (`localhost:5000/`) in a browser, and the following happens:
    * The browser connects to the Plack server, an instance of `HTTP::Server::PSGI`, and issues a GET request for the Scraping program. The single slash (`/`) at the end of the URL identifies this program. (A modern browser would add the closing slash even if the user failed to do so.)
    * The Scraping program then issues a GET request for the data.html document. If the request succeeds, the application extracts the relevant data from the document using the `HTML::TableExtract` package, saves the extracted data to a file, and takes some basic statistical measures that represent processing the extracted data. An HTML report like the following is returned to the user's browser.


![HTML report generated by the Scraping program][3]

Fig. 1: Final report from the Scraping program

The request traffic from the user's browser to the Plack server and then to the server hosting the data.html document (e.g., Nginx) can be depicted as follows:
```
              GET localhost:5000/             GET localhost:80/data.html

user's browser------------------->Plack server-------------------------->Nginx

```

The final step involves only the Plack server and the user's browser:
```
             reportFinal.html

Plack server------------------>user's browser

```

Fig. 1 above shows the final report document.

### The scraping program in detail

The source code and data file (data.html) are available from my [website][4] in a ZIP file that includes a README. Here is a quick summary of the pieces, and clarifications will follow:
```
data.html             ## data source to be hosted by a web server

scrape.pl             ## main source code, run with the plackup utility (see below)

Stats::Controller.pm  ## handles request routing, data extraction, and processing

Stats::Util.pm        ## utility functions used in Controller.pm

report.html           ## HTML template used to generate the report

rawData.dat           ## the extracted data

```

The `Plack/PSGI` packages come with a command-line utility named `plackup`, which can be used to launch the Scraping program. With `%` as the command-line prompt, the command for starting the Scraping program is:
```
% plackup scrape.pl

```

The `plackup` command starts a standalone Plack web server that hosts the Scraping program. The Scraping code handles request routing, extracts data from the data.html document, produces some basic statistical measures, and then uses the `Template::Recall` package to generate an HTML report for the user. Because the Plack server runs indefinitely, the Scraping program prints the process ID, which can be used to kill the server and the Scraping app.

`Plack/PSGI` supports Rails-style routing in which an HTTP request is dispatched to a specific request handler based on two factors:

  * The HTTP request method (verb) such as GET or POST.
  * The Uniform Resource Identifier (URI or noun) for the requested resource; in this case the standalone finishing slash (`/`) in the URL `http://localhost:5000/` that a user enters in a browser once the Scraping program has launched.



The Scraping program handles only one type of request: a GET for the resource named `/`, and this resource is the screen-scraping and data-processing code in my `Stats::Controller` package. Here, for review, is the `Plack/PSGI` routing setup, right at the top of source file scrape.pl:
```
my $router = router {

    match '/', {method => 'GET'},   ## noun/verb combo: / is noun, GET is verb

    to {controller => 'Controller', action => 'index'}; ## handler is function get_index

    # Other actions as needed

};

```

The request handler `Controller::get_index` has only high-level logic, leaving the screen-scraping and report-generating details to utility functions in the Util.pm file, as described in the following section.

### The screen-scraping code

Recall that the Plack server dispatches a GET request for `localhost:5000/` to the Scraping program's `get_index` function. This function, as the request handler, then starts the job of retrieving the data to be scraped, scraping the data, and generating the final report. The data-retrieval part falls to a utility function, which uses Perl's `LWP::Agent` package to get the data from whatever server is hosting the data.html document. With the data document in hand, the Scraping program invokes the utility function `extract_from_html` to do the data extraction.

The data.html document happens to be well-formed XML, which means a Perl package such as `XML::LibXML` could be used to extract the data through an explicit XML parse. However, the `HTML::TableExtract` package is inviting because it bypasses the tedium of XML parses, and (in very little code) delivers a Perl hash with the extracted data. Data aggregates in HTML documents usually occur in lists or tables, and the `HTML::TableExtract` package targets tables. Here are the three critical lines of code for the data extraction:
```
my $col_headers = col_headers(); ## col_headers() returns an array of the table's column names

my $te = HTML::TableExtract->new(headers => $col_headers);

$te->parse($page);  ## $page is data.html

```

The `$col_headers` refers to a Perl array of strings, each a column header in the HTML document:
```
sub col_headers {    ## column headers in the HTML table

    return ["Area",

            "MedianWage",

            ...

            "BoostFromGradDegree"];

}col_headers

```

After the call to the `TableExtract::parse` function, the Scraping program uses the `TableExtract::rows` function to iterate over the rows of extracted data—rows of data without the HTML markup. These rows, as Perl lists, are added to a Perl hash named `%majors_hash`, which can be depicted as follows:

  * Each key identifies an area of study such as Computing or Engineering.

  * The value of each key is the list of seven extracted data items, where seven is the number of columns in the HTML table. For Computing, the list with annotations is:
```
        name            median  % with this degree  income boost from GD
        /                 /            /            /
    (Computing  55000  75000  112000  5.1%  32.0%  31.0%)   ## data items
                 /              \           \
           25th-ptile      75th-ptile  % going on for GD = grad degree
```




The hash with the extracted data is written to the local file rawData.dat:
```
ForeignLanguage 50000 35000 75000 3.5% 54% 101%
LiberalArts 47000 32000 70000 9.7% 41% 48%
...
Engineering 78000 54000 104000 8.2% 37% 32%
Computing 75000 51000 112000 5.1% 32% 31%
...
PublicPolicy 50000 36000 74000 2.3% 24% 45%
```

The next step is to process the extracted data, in this case by doing rudimentary statistical analysis using the `Statistics::Descriptive` package. In Fig. 1 above, the statistical summary is presented in a separate table at the bottom of the report.

### The report-generation code

The final step in the Scraping program is to generate a report. Perl has options for generating HTML, and `Template::Recall` is among them. As the name suggests, the package generates HTML from an HTML template, which is a mix of standard HTML markup and customized tags that serve as placeholders for data generated from backend code. The template file is report.html, and the backend function of interest is `Controller::generate_report`. Here is how the code and the template interact.

The report document (Fig. 1) has two tables. The top table is generated through iteration, as each row has the same columns (area of study, income for the 25th percentile, and so on). In each iteration, the code creates a hash with values for a particular area of study:
```
my %row = (
     major => $key,
     wage  => '$' . commify($values[0]), ## commify turns 1234 into 1,234
     p25   => '$' . commify($values[1]),
     p75   => '$' . commify($values[2]),
     population   => $values[3],
     grad  => $values[4],
     boost => $values[5]
);

```

The hash keys are Perl [barewords][5] such as `major` and `wage` that represent items in the list of data values extracted earlier from the HTML data document. The corresponding HTML template looks like this:
```
[ === even  === ]
<tr class = 'even'>
   <td>['major']</td>
   <td align = 'right'>['p25']</td>
   <td align = 'right'>['wage']</td>
   <td align = 'right'>['p75']</td>
   <td align = 'right'>['pop']</td>
   <td align = 'right'>['grad']</td>
   <td align = 'right'>['boost']</td>
</tr>
[=== end1 ===]
```

The customized tags are in square brackets. The tags at the top and the bottom mark the beginning and the end, respectively, of a template region to be rendered. The other customized tags identify individual targets for the backend code. For example, the template column identified as `major` matches the hash entry with `major` as the key. Here is the call in the backend code that binds the data to the customized tags:
```
print OUTFILE $tr->render('end1');

```

The reference `$tr` is to a `Template::Recall` instance, and `OUTFILE` is the report file reportFinal.html, which is generated from the template file report.html together with the backend code. If all goes well, the reportFinal.html document is what the user sees in the browser (see Fig. 1).

The scraping program draws from excellent Perl packages such as `Plack/PSGI`, `LWP::Agent`, `HTML::TableExtract`, `Template::Recall`, and `Statistics::Descriptive` to deal with the often messy task of screen-scraping for data. These packages play together nicely, as each targets a specific subtask. Finally, the Scraping program might be extended to cluster the extracted data: The `Algorithm::KMeans` package is suited for this extension and could use the data persisted in the rawData.dat file.

--------------------------------------------------------------------------------

via: https://opensource.com/article/18/6/screen-scraping

作者：[Marty Kalin][a]
选题：[lujun9972](https://github.com/lujun9972)
译者：[译者ID](https://github.com/译者ID)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]:https://opensource.com/users/mkalindepauledu
[1]:https://en.wikipedia.org/wiki/Data_scraping#Screen_scraping
[2]:/file/399886
[3]:https://opensource.com/sites/default/files/uploads/scrapeshot.png (HTML report generated by the Scraping program)
[4]:http://condor.depaul.edu/mkalin
[5]:https://en.wiktionary.org/wiki/bareword