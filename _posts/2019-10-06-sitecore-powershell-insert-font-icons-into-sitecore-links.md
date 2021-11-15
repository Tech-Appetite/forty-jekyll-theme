---
layout: post
title: 'Sitecore PowerShell Series: Insert appropriate font icons into Sitecore Links with Html Agility Pack!'
description: Learn how to manipulate Sitecore data from Rich-Text Editor fields with Sitecore PowerShell in order to add different types of font icons to Sitecore Links
image: assets/images/sitecore-and-powershell.jpg
---

## The Scenario
You just completed a fairly substantial content migration from an archaic CMS into Sitecore. These new Product Detail Pages all contain associated content items labeled "Support Links" that holds a single Rich-Text Editor field that simply renders on the page as a Datasource. The client takes one quick look at the page and notices that all of the support links are missing super important icons that are used to indicate the file type of PDF or Zip. These icons were clearly in the new design, but this old data never had the markup required.

Let's take a look at the old markup and the desired markup:

_old markup_
```html
<some_html>
    <a href="~/media/[guid].ashx">Support Link 1 (Pdf)</a>
    <a href="~/media/[guid].ashx">Support Link 2 (Zip)</a>
</some_html>
```
_new markup_
```html
<some_html>
    <a href="~/media/[guid].ashx">
        <em class='fa fa-file-pdf-o' aria-hidden='true' style='padding-right: 5px;'></em>
        Support Link 1 (Pdf)
    </a>
    <a href="~/media/[guid].ashx">
        <em class='fa fa-file-archive-o' aria-hidden='true' style='padding-right: 5px;'></em>
        Support Link 2 (Zip)
    </a>
</some_html>
```
Doing this manually would be exhausting. We would have to not only go through every content item of this type and add the `em` markup, but also parse the link's `Dynamic url` to figure out if it's a Pdf or a Zip.
<img src="{% link assets/images/i_hug_that_feel.png %}" alt="" />

## Luckily for us..
We have PowerShell! And because we're working in context of the ISE, we can already reference the Html Agility Pack included with Sitecore (I've yet to come across any issues with the version being from _2014_ even in Sitecore v9.1.1, but you may find yourself using a newer method that can be resolved by loading a newer dll with `[Reflection.Assembly]::LoadFile`).

We can start the script with getting all the items named "Support Links" with a specific template id:
```powershell
$supportItems = Get-Item -Path master: -Query "/sitecore/content/Tenant/Site A/Homepage//*[@@templateid='{A099DC2D-1E23-499F-B101-DBB0902148F4}' and @@name='Support Links']"
```

Loop through these items and load up the content from the Rich-Text Editor field into the Agility Pack magic:
```powershell
$supportItems | ForEach-Object {
    $supportItem = $_
    $content = $_."Content"    ## accessing the field value this way lets us not worry about the Begin/End Edit requirements
    
    $htmlDocument = New-Object -TypeName HtmlAgilityPack.HtmlDocument
    $htmlDocument.LoadHtml($content)
```

Find all the anchors using HtmlAgilityPack XPath query:
```powershell
foreach($x in $htmlDocument.DocumentNode.SelectNodes("//a")) {    ## foreach anchor in html
    $href = $x.Attributes["href"].Value
```

Check if the anchor is a Sitecore _Dynamic Url_, capture the Guid and retrieve the Media Item:
```powershell
if ($href  -like "~/media/*") {
    $guid = New-Object -TypeName System.guid -ArgumentList $([System.IO.Path]::GetFileNameWithoutExtension($href))        ## parse out the guid with a C# Path Helper Method!
    $mediaItem = Get-Item -Path master: -ID "{$($guid)}"
}
```

If the `Media Item` is a Zip or Pdf AND the icon has yet to be added, adjust the anchor's inner HTML to include the icon:
```powershell
if ($mediaItem.TemplateName -eq "Pdf" -and (-not $x.InnerHtml.Contains("fa-file-pdf"))) {
    Write-Host "Found PDF Link for Link [$($href)], DisplayName [$($mediaItem.DisplayName)]"								## LOG some good info
    $x.InnerHtml = "<em class='fa fa-file-pdf-o' aria-hidden='true' style='padding-right: 5px;'></em>" + $x.InnerHtml		## append the icon
}
elseif ($mediaItem.TemplateName -eq "Zip" -and (-not $x.InnerHtml.Contains("fa-file-archive"))) {
	Write-Host "Found Zip Link for Link [$($href)], DisplayName [$($mediaItem.DisplayName)]"
	$x.InnerHtml = "<em class='fa fa-file-archive-o' aria-hidden='true' style='padding-right: 5px;'></em>" + $x.InnerHtml
}
```

Now that we've made our updates in the `$htmlDocument` object, we can take that HTML and put it back in the Sitecore field:
```powershell
$newHTML = $htmlDocument.DocumentNode.OuterHtml
$_."Content" = $newHTML
```

## Bonus Round
So you let the content authors in to the system a little too early and, lo and behold, over 100 content items were updated to include an inline-style attribute `color:green` on the icons. I mean, you gotta hand it to them for trying to fix the design, right? Fortunately you know that if you were to remove this inline-style, the anchor tag's _branded_ color would be inherited from the inner icon tag once again. Back to the script board we go.

Within the same inner foreach anchor tag, grab the font tag that can either be an `<em>` or an `<i>`:
```powershell
$nodes_em = @($x.ChildNodes["em"])		## fun tip: default to an array, @(), since ChildNodes can condense down to returning a single item
$nodes_i = @($x.ChildNodes["i"])
```

If any nodes are found with HTML and contains the keyword "green", remove this style using PowerShell's `IndexOf` and `Remove` String methods:
```powershell
if ($nodes_i.Count -and $nodes_i[0].OuterHtml -and -not [System.String]::IsNullOrWhiteSpace($nodes_i[0].OuterHtml) -and $nodes_i[0].OuterHtml.Contains("green")) {
	Write-Host "Found color:green to remove from <i>: [$($href)] [$($pcItem.Id)] [$($nodes_i[0].OuterHtml)]"		## LOG the icon tag html to be removed
	
	$attr = $nodes_i[0].Attributes["style"]			## grab the inline-style attribute value
	
	$idxStart = $attr.Value.IndexOf("color:")		## get the start and end indexes of the color attribute in order to remove it
	$idxEnd = $attr.Value.IndexOf(";", $idxStart)
	
	if ($idxEnd -gt $idxStart) {					## always check to make sure the end index was set, since some inline-styles don't end with a semi-colon
	   $attr.Value = $attr.Value.Remove($idxStart, $idxEnd - $idxStart + 1)
	   Write-Host "Removed inline-style color attribute, new inline-style value [$($attr.Value)]"
	}
}
```

In this walkthrough we learned how to manipulate existing HTML within the Sitecore CMS with the help of Html Agility Pack and some small C# methods. As always, make sure to set `return` in loops and `exit` before making changes that could make permanent incorrect changes to your environment! And if you're looking to bulk update more than 260 items, look to switch to the `Find-Items` cmdlet that uses Sitecore Indexes.