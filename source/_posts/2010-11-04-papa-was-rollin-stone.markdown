---
layout: post
title: Papa Was a Rollin' Stone
date: 2010-11-04 00:15
comments: true
categories: Fun
---

I don't know for sure if it was a joke or not, but while investigating the source code of John Papa's recent Silverlight PDC demo, I've found some really funny (and weird :) stuff. Just take a look.
<!-- more -->

{% codeblock lang:csharp %}
public class Core
{
	public const string StateKeyParameter = "statekey";

	public static string NewKey()
	{
		return Guid.NewGuid().ToString().RemoveChar('-');
	}
}
{% endcodeblock %}

Hmmm, this can be done more optimally. There is no need to remove hyphens from canonical string representation of `Guid`. The `ToString()` method accepts standard format specifier and according to MSDN one may just specify "N" in order to get `Guid` representation in the form of "00000000000000000000000000000000".

But the actual fun started once I've recognized, that I can't recall I've ever seen `RemoveChar` method before. The `String` class have no such method. The `RemoveChar` is an extension method defined as follows:

{% codeblock lang:csharp %}
public static class StringExtensions
{
	public static string RemoveChar(this string text, char character)
	{
		string[] parts = text.Split(character);
		var sb = new StringBuilder();
		foreach (var part in parts)
		{
			sb.Append(part);
		}
		return sb.ToString();
	}
}
{% endcodeblock %}

:) hehe, funny, no?
