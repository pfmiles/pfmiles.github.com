---
layout: page
title: "My Gists"
date: 2014-04-16 17:15
comments: false
sharing: true
footer: true
---

<script type="text/javascript" src="http://code.jquery.com/jquery-2.1.0.min.js">
</script>
<script type="text/javascript">
$.ajax({
         "type":"GET",
         "url":"https://api.github.com/users/pfmiles/gists",
         "data":"",
         "success":function(data){
                var ctt = "";
                for(index in data){
                    var gist = data[index];
                    ctt += '<a href="'+ gist.html_url +'">' + gist.description + '</a><br/><br/>';
                }
                $("p#gists").html(ctt);
             }
    });
</script>
<span>Ctrl + F to search...</span>
<p id="gists"></p>
