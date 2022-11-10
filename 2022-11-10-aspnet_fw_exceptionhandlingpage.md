
---
title: ASP.NET MVC Framework - Exceptionhandling page
date: 2022-11-10
---

# ASP.NET MVC Framework - Building an Exceptionhandling PAge

Inside the `Global.asax` / `HttpApplication`

~~~csharp
protected void Application_Error(Object sender, EventArgs e)
{
    Exception exception = Server.GetLastError();
    Server.ClearError();

    var routeData = new RouteData();
    routeData.Values.Add("controller", "ErrorPage");
    routeData.Values.Add("action", "Error");
    routeData.Values.Add("exception", exception);
    routeData.Values.Add("url", HttpContext.Current.Request.Url.ToString());

    if (exception.GetType() == typeof(HttpException))
    {
        var httpException = (HttpException)exception;
        routeData.Values.Add("statusCode", httpException.GetHttpCode());
    }
    else
    {
        routeData.Values.Add("statusCode", 500);
    }

    Response.TrySkipIisCustomErrors = true;
    IController controller = new ErrorPageController();
    controller.Execute(new RequestContext(new HttpContextWrapper(Context), routeData));
    Response.End();
}
~~~

Then introduce Controller for the page
~~~csharp
public class ErrorPageController : Controller
{
    public ActionResult Error(int statusCode, Exception exception, string url)
    {
        Response.StatusCode = statusCode;
        ViewBag.StatusCode = statusCode + " Error";
        return View(new ErrorInfo()
        {
            HttpStatusCode = statusCode,
            Exception = exception,
            Url = url
        });
    }
}
~~~

At last create the page `Views/Shared/Error.cshtml`
~~~html
@model Post.SysFN.Web.Models.ErrorInfo

@{
    ViewBag.Title = "Error";
}

<h1 class="text-danger">Ein Fehler ist während der Verarbeitung der Anfrage aufgetreten.</h1>
<p>Der Fehler trat auf der URL <a href="@Model.Url">@Model.Url</a> auf, die Antwort gab den Statuscode <b>@Model.HttpStatusCode</b> zurück</p>

@if (Html.IsDebug())
{
    <div>
        <h3>Exception Message</h3>
        <p>@Model.Exception.Message</p>
        <h3>Exception Stack Trace</h3>
        <p style="white-space: pre-line">@Model.Exception.StackTrace</p>
    </div>
}

<h6>@Html.ActionLink("Go Back To Home Page", "Index", "Home")</h6>
~~~ 

At least in the web.config
~~~xml
<configuration>
   <system.webServer>
            <httpErrors existingResponse="PassThrough"></httpErrors>
        </system.webServer>  
</configuration>

~~~

[^1]:https://stackoverflow.com/questions/21679648/mvc-error-page-controller-and-custom-routing]
[^2]:https://stackoverflow.com/questions/6033681/how-to-override-httperrors-section-in-web-config-asp-net-mvc3
[^3]:https://stackoverflow.com/a/29712033
