---
layout: page
title: Our Team
permalink: /team/
---

<style type="text/css">
	.teamImage {
		padding: 10px;
		width: 30%;
		display: inline-block;
	}
	.profile {
		text-align: center;
	}
</style>

<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-91132149-1', 'auto');
  ga('send', 'pageview');

</script>

{% for member in site.data.members %}
  <div class="teamImage">
    <img style="border-radius: 50%" src="{{site.url}}/images/team/{{member.image}}">
    <div class="profile">
		<span>{{member.name}}</span><br>
		<span style="font-style: italic; color: #82827A">
			{{member.title}}
			{% if member.linked-in != "" %}
				<a href="{{member.linked-in}}"><img style="width: 15px; height: 15px" src="{{site.url}}/images/linked-in-logo.png"></a>
			{% endif %}
		</span>
    </div>
  </div>
{% endfor %}
