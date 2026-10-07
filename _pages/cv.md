---
layout: page
permalink: /cv/
title: CV
nav: false
nav_order: 5
cv_pdf: /assets/pdf/cv.pdf # you can also use external links here
description: My CV as a PDF. If the embedded viewer does not load, use the download link.
---

<p>
  <a class="btn btn-sm z-depth-0" role="button" href="{{ page.cv_pdf | relative_url }}" target="_blank" rel="noopener noreferrer">
    <i class="fa-solid fa-file-pdf"></i> Download PDF
  </a>
</p>

<object data="{{ page.cv_pdf | relative_url }}" type="application/pdf" width="100%" style="min-height: 85vh;">
  <p>Your browser cannot display the PDF inline. <a href="{{ page.cv_pdf | relative_url }}">Download the CV</a> instead.</p>
</object>
