# Using Puppeteer (headless chrome) in an Azure Web App

Our team recently started running into issues with our HTML to PDF conversion. We've been using PhantomJS, and upon discovering that it is no longer supported and the issues we've been having weren't going to go away, we realised we'll be needing a new solution. After some research, all signs pointed to headless chrome. We know it has a solid rendering engine behind it, and it has good support for custom headers and footers - which is a requirement for our application.

Unfortunately, all of our research somehow didn't include whether Azure - where we host all our apps - actually supports a headless browser. And ofcourse, to our horror, when we deployed to staging it wasn't working. Having spent all that time on the migration, however, we definitely weren't going to give up there. So after a couple more days of research and effort we got it working, and I'm hoping this guide will save you some time in doing the same.

Just to clarify, this guide is going to describe how we got puppeteer running for the purpose of generating a PDF from HTML in a .NET Core Azure Web App.

### A summary of the solution

What we ended up doing was having headless chrome run in a Docker container, in an Azure Function, which gets called via an HTTP request from the web app.

Essentially this is what goes down:
Web App ---HTML--> Azure Function running in Docker Container ---PDF--> Back to Web App

Disclaimer: This was my first time working with a Docker Container, and my knowlegde on the matter is limited to literally only what's required to get this solution to work.

### Step 1: Getting a Azure Function Docker Image running and working locally

This in itself is already a multi-step process, so I'm just going to link you to the guide I used to complete this step:
https://medium.com/@nahuelfedericoberg/azure-functions-docker-basic-demo-4a01be1c15bd
In step 3 of the guide (creating the new function), be sure to select javascript (node) instead of C#, and then select HttpTrigger.
If you're running on Windows and you're having trouble connecting to the function on localhost, you can try connecting straight to the Docker Machine's IP instead (this is what I had to do).

### Step 2: Altering the Azure Function to do HTML to PDF conversion

Here we first have to get Puppeteer involved. Add the following to your dockerfile:

    FROM estruyf/azure-function-node-puppeteer
    
    COPY . /home/site/wwwroot

This will use the following project to make puppeteer available:
https://github.com/estruyf/azure-function-node-puppeteer

Now for the actual function itself. Here's the code that I used:

```javascript
const puppeteer = require('puppeteer');

module.exports = async (context, req) => {    
    const browser = await puppeteer.launch({
        args: [
            '--no-sandbox',
            '--disable-setuid-sandbox'
        ]
    });

    const pageBody = req.body.pdfBody;
    const headerHtml = req.body.headerHtml;
    const footerHtml = req.body.footerHtml;
    const headerHeight = req.body.headerHeight;
    const footerHeight = req.body.footerHeight;

    const pdfOptions = {
        scale: 1,
        displayHeaderFooter: true,
        printBackground: true,
        format: "A4",
        margin: {
            top: headerHeight,
            bottom: footerHeight,
            right: "0px",
            left: "0px",
        },
        headerTemplate: headerHtml,
        footerTemplate: footerHtml
    }

    try {

        const page = await browser.newPage();
        await page.goto("data:text/html," + pageBody, {waitUntil: "networkidle0"});

        const pdf = await page.pdf(pdfOptions);

        await browser.close();

        context.res = {
            // status: 200, /* Defaults to 200 */
            body: pdf
        };

    }
    catch (e) {
        context.log(e);
        context.res = {
            status: 500
        }
    }
};
```

This function gets triggered by an HTTP request, and we receive the HTML in the request body. It expects a JSON object with the following properties:
pdfBody:      html string
headerHtml:   html string
footerHtml:   html string
footerHeight: string (e.g. "100px")
headerHeight: string (e.g. "100px")

You can ofcourse play around with the PDF options and the above object if you don't require the header or footers. We return the PDF as a byte array in the HTTP response body.

### Step 3: Deploy the docker image to an Azure Function

This is also a multi-step process, so I will again just link to the guide I used to accomplish this. It involves setting up a Docker Hub repository, pushing the image to that repo, then linking up to it from the newly created Azure Function. Here's the guide:
https://medium.com/@carlosboichu87/using-azure-functions-with-docker-a3bb5da7526a
You can just skip to the "Deploying an Azure Function using a Docker Container. (in Azure Portal)" section.

### Step 4: Calling the function from our .net Core code
I'll again give you the code then run you through it:
```c#
public async static Task<byte[]> PDFCall(string pdfBody, string headerHtml, string footerHtml, string headerHeight, string footerHeight)
{
  HttpClient client = new HttpClient();
  client.BaseAddress = new Uri("https://[YOUR_FUNCTION].azurewebsites.net/api/[YOUR_FUNCTION_NAME]");
  client.DefaultRequestHeaders.Accept.Clear();
  client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

  var pdfObj = new {
    pdfBody = pdfBody,
    headerHtml = headerHtml,
    footerHtml = footerHtml,
    headerHeight = headerHeight,
    footerHeight = footerHeight
  };

  var content = new StringContent(JsonConvert.SerializeObject(pdfObj), Encoding.UTF8, "application/json");

  HttpResponseMessage response = client.PostAsync("", content).Result;

  if (!response.IsSuccessStatusCode)
  {
    throw new Exception("Could not use PDF API");
  }

  return await response.Content.ReadAsByteArrayAsync();
}
```
Its pretty self-explanatory. We set up the object to be sent in the request body, then we make the request with the serialized object. If the response is successful, we return the response as a byte array. That's our PDF! Do with it as you please.

### Conclusion

I hope this has been helpful for some readers. If you have any suggestions to improve this guide, please feel free.

