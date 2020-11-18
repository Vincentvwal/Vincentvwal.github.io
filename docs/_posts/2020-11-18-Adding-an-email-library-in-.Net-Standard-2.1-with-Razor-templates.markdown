---
layout: post
title:  "Adding an email library in .Net Standard 2.1 with Razor templates."
date:   2020-11-18 17:10:26 +0100
categories: RazorLight Templates Razor Email
---
As we are moving to a microservice architecture, we require a new solution for sending out emails to our customers. Currently this is done by using a .Net Framework 4.6.1 library, however since we are using .Net Core 3.1 we are required to use a .Net Standard 2.1 class library.

However because of the changes in the .Net Standard framework, we are by default not able to use Razor templates (which we are already using) and to simplify this move we wanted to keep our current templates which saves the time of re-writing these.

Lets start by creating a new solution in Visual Studio. For this example i will use a .Net Core Console application named Email.Core. Next up, also create a .Net Standard class library targeted for 2.1 named Email.Templates.

Within the Templates project add the RazorLight nuget package using the package manager console, at the time of writing we are using version 2.0.0-beta7. This will allow us to use our razor templates.

{% highlight C# %}
Install-Package RazorLight -Version 2.0.0-beta7
{% endhighlight %}

Next up, create 2 folders in the root, named Templates & Models. Templates will hold our Razor templates, where models will hold our models which are used within the razor templates.

Next up, create an interface in the root called IAmAssembly. Even though this interface will hold nothing except for the interface spec itself., this will be used as an identifier for RazorLight to see in which assembly our templates will live.

Now add a new class named ParseableEmailTemplate which will serve as our class which holds the unparsed template and the name of it.
Add the following code to this

{% highlight C# %}
internal class ParseableEmailTemplate
{
	public string UnparsedTemplate { get; private set; }
	public string Name { get; private set; }
	public ParseableEmailTemplate(string unparsedTemplate, string name)
	{
		UnparsedTemplate = unparsedTemplate;
		Name = name;
	}
}
{% endhighlight %}

Next up, create new interface named ITemplateParser and add it’s class as well. This will be our main function which will be used to get the templates, cache them (well, RazorLight will do that) and parse them to html strings.
Within this interface, add the following code, we will write the implementation for this at a later time.

{% highlight C# %}
Task<string> ParseTemplate<T>(T emailModel);
{% endhighlight %}

Now within the TemplateParser, add the following code at the top of the class.

{% highlight C# %}
private readonly string _templateFolderPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "Templates");
{% endhighlight %}

This will serve as our identified where our templates are.
Next up, add the following GetTemplates function.

{% highlight C# %}
internal ICollection<ParseableEmailTemplate> GetTemplates()
{
	List<ParseableEmailTemplate> templates = new List<ParseableEmailTemplate>();

	/* If no templates directory exist, return nothing */
	var templateDirectories = new DirectoryInfo(_templateFolderPath);
	if (!templateDirectories.Exists)
		return new List<ParseableEmailTemplate>();

	var files = templateDirectories.GetFiles();
	foreach (var file in files)
	{
		var template = new ParseableEmailTemplate(File.ReadAllText(file.FullName), file.Name.Split('.')[0]);
		templates.Add(template);
	}

	return templates;
}
{% endhighlight %}

This will run trough all files in the template directory and add all templates to the list, after which it returns them. There is a check to see if the directory exists, if it does not, return an empty list.

Now that we have our list of templates, we are required to compile these into a RazorLight spec. To do this we will use the CompileRenderStringAsync function of Razorlight. But first we need to add RazorLight to our TemplateParser, do this by adding the following.

{% highlight C# %}
internal readonly RazorLightEngine razorEngine;
{% endhighlight %}

Now we can add the function to compile our functions, do this by adding the following function.

{% highlight C# %}
internal async Task CompileTemplates()
{
	var templates = GetTemplates();
	foreach (var template in templates)
	{
		var typeName = string.Concat("Email.Templates.Models.", template.Name);
		var type = Type.GetType(typeName);
		var instance = Activator.CreateInstance(type);

		await razorEngine.CompileRenderStringAsync(typeName, template.UnparsedTemplate, instance);
	}
}
{% endhighlight %}

This will get each template and it’s associated model and compile this for later use. As you can see we call the GetTemplates from this function, as this is the only place we will use it.

Now, create the constructor which is required to instantiate both the templates and RazorEngine by using Dependency Injection.

{% highlight C# %}
public TemplateParser()
{
	razorEngine = new RazorLightEngineBuilder().UseEmbeddedResourcesProject(typeof(IAmAssembly).Assembly).UseMemoryCachingProvider().Build();
	CompileTemplates().GetAwaiter().GetResult();
}
{% endhighlight %}

Some magic happens here, we create a new instance of razorEngine and we say it needs to use embedded resources within the project of IAmAssembly. Finally we tell it to use Memory Caching to prevent the templates from being compiled each call which saves both memory and time :). Finally we compile all templates and place them in the RazorEngine.

Now we can implement our template parser we created earlier, do this by adding the following code.

{% highlight C# %}
public async Task<string> ParseTemplate<T>(T emailModel)
{
	var template = await razorEngine.CompileTemplateAsync(emailModel.GetType().FullName);
	var content = await razorEngine.RenderTemplateAsync(template, emailModel);

	return content;
}
{% endhighlight %}

Note that we try to compile the template again, however, since the template is already in memory this will not happen, instead it will return a template of type ITemplatePage which is required for the render function. In the render function we pass the template and the model which will return our parsed and fully filled html content 🙂

Now all that is left is add a simple model

{% highlight C# %}
namespace Email.Templates.Models
{
    public class HelloWorld
    {
        public string Name { get; set; }
        public DateTime Date { get; set; }
    }
}
{% endhighlight %}

And a simple template

{% highlight C# %}
@model Email.Templates.Models.HelloWorld
<html>
<body>
    <table>
        <tr>
            <td width="580" height="64" style="font-size: 1px; background-color: #f2f2f2">&nbsp;</td>
        </tr>
        <tr>
            <td class="fullcol" width="580" align="center" style="padding: 0 4% 0 4%; background-color: #f2f2f2; font-family: tahoma,sans-serif;">
                Hello @Model.Name at @Model.Date
            </td>
        </tr>
    </table>
</body>
</html>
{% endhighlight %}

After which we can test it in the following way

{% highlight C# %}
static void Main(string[] args)
{
	ITemplateParser _templateParser = new TemplateParser();

	var model = new HelloWorld();
	model.Name = "Vincent";
	model.Date = DateTime.Now;

	var htmlContent = _templateParser.ParseTemplate(model).GetAwaiter().GetResult();

	Console.WriteLine(htmlContent);
	Console.ReadKey();
}
{% endhighlight %}