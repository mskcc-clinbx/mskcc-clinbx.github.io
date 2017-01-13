---
layout: page
title: Our Team
permalink: /team/
---

<style type="text/css">
	.teamImage {
		margin: 10px;
		width: 30%;
		display: inline-block;
	}
	.profile {
		text-align: center;
	}
</style>

{% for member in site.data.members %}
  <div class="teamImage">
    <img style="border-radius: 50%" src="{{site.url}}/images/team/{{member.image}}">
    <div class="profile">
		<span>{{member.name}}</span><br>
		<span style="font-style: italic; color: #82827A">
			{{member.title}} 
			{% capture linked-in-link %}
				{{member.linked-in}}
			{% endcapture %}
			{% if linked-in-link %}
				<a href="{{member.linked-in}}"><img style="width: 15px; height: 15px" src="{{site.url}}/images/linked-in-logo.png"></a>
			{% endif %}
		</span>
    </div>
  </div>
{% endfor %}