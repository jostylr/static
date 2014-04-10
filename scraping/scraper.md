# Getting out of an old site

This is how to get the content out of a legacy site. For example, you might
have a blog using Wordpress or on tumblr. The program below should help in
getting your content out of the old and into markdown format. 

This literate program will do that. 

It may require some tweaking. Some confidence in technical matters is
required. 

Please note that you should only use this for content you own. By default, the
maximum number of pages scraped is 3. This should be thought of as a test run.
After a successful test, you will probably need to increase the size. 

* [spider.js](#specs "save: |jshint")


## Specs

So we need to webscrape. We load up the page, grab the content of
interest, get links for the next one, through the content into a html to
markdown script, compile a list of the pages. 

The first libraries I will attempt to use is [jsdom](https://npmjs.org/package/jsdom) for both web scraping and grabbing the content. 

The other library will be [html-md](https://npmjs.org/package/html-md) which I hope works well. It uses jsdom to use the living document to translate it. 


    var jsdom = require('jsdom');
    var md = require('html-md');
    var fs = require('fs');
    var http = require('http');
    var cheerio = require("cheerio");

Initial setup. Set the url to where you want to start from. It will store the
pages in pages with full url as key and the [fname, title, content, raw] as
the page. The links still to be done are stored in waiting.

    var counter = 0;
    var max = 3;
    var root = "www.bohmianmechanics.org";
    var content = "#content";
    var dir = "pages";
    var pages = {};
    

    var fetcher, html, other, writecb, done;

    fetcher = _"fetcher";
    
    html = _"scraper";
   
    other = _"other";

This does not do much. There really shouldn't be errors and we don't really
care about when it is done. 


    writecb = function (err) {
                if (err) {
                    throw err;
                }
            };

This function is called when all the pages are done being scraped. 


    done = function () {
        var list = [], 
            page, url, fname;
        for (url in pages) {
            fname = url.slice(root.length + 1);
            page = pages[url]; 
            fs.writeFile(fname, page.title + "\n\n" + 
                page.content.trim(), writecb);
            list.push(fname);
        }
        fs.writeFile("list.txt", list.join("\n"), writecb);

    };

    console.log("starting", root);

    fetcher("/", html); 


## Scraper

This runs jsdom to scrape, grabs al the goodies and then either quits or gets another, depending.

    function (path, body) {
        var $ = cheerio.load(body);
        jsdom.env(body, function (errors, window) {
            var doc = window.document;


Grab the title and the main content, transforming it into markdown. 

            var obj = {
                path : path,
                title :
                doc.getElementsByTagName("title")[0].textContent.trim() ,
                content : md($(content).html()),
                raw : doc.children[0].outerHTML
            };

Figure out filename path by slicing from the end of the root. This could be
dangerous. 

            var fname = path;
            if ( fname[fname.length-1] === "/" ) {
                fname = fname + "index";
            }
            console.log(fname);
            if ( fname.indexOf("..") !== -1 ) {
                console.log("bad filename", fname);
                fname = "bad";
            } else {
                if (fname.match(/\.(html|htm)$/) ) {
                    fname = fname.replace(/\.(html|htm)$/, ".md");
                } else {
                    fname = fname + ".md";
                }
                console.log(obj.title + "\n\n" + 
                obj.content.trim());

                fs.writeFile(dir + fname, obj.title + "\n\n" + 
                obj.content.trim(), writecb); 
                console.log("writing", fname);
            }
            
            obj.fname = fname;

            pages[path] = obj;

Grab all links and queue them. 

           var links =  Array.prototype.slice.call(doc.getElementsByTagName("a") ) ;
           $("a").each(function (i, el) {
                if (counter > max) {
                    return;
                }
                console.log($(el).attr("href"));
                var href = $(el).attr("href");
                var index = href.indexOf(root);
                if (index !== -1) {
                    href = href.slice(index+root.length);
                    if ( !(pages.hasOwnProperty(href)) ) {
                        pages[href] = "";
                        if ( href.match(/(\/|\.html|\.htm)$/) ) {
                            counter += 1;
                            fetcher(href, html);
                        } else if (!(pages.hasOwnProperty(href) ) ) {
                            counter += 1;
                            fetcher(href, other);
                        }
                    }
                }
           });
        });

    }

## Fetcher

Basic web fetcher. Note we read it all. Grabbed from http://stackoverflow.com/questions/9577611/http-get-request-in-node-js-express 

    function (path, cb) {

        var options = {
          host: root,
          path: path
        };

        console.log(options);

        var req = http.get(options, function(res) {
          //console.log('STATUS: ' + res.statusCode);
          //console.log('HEADERS: ' + JSON.stringify(res.headers));

          // Buffer the body entirely for processing as a whole.
         var bodyChunks = [];
          res.on('data', function(chunk) {
            // You can process streamed parts here...
            bodyChunks.push(chunk);
          }).on('end', function() {
            var body = Buffer.concat(bodyChunks);
            //console.log('BODY: ' + body);
            // ...and/or process the entire body here.
            cb(path, body.toString() ); 
          });
        });

        req.on('error', function(e) {
          console.log('ERROR: ' + e.message);
        });

    }

## Other

This saves anything that is not an html file. 

    function (path, text) {
        if (path && (path.indexOf("..") === -1) ) {
            fs.writeFile(dir+path, text);
        } else {
            console.log("BAD", path, text);
        }
    }

