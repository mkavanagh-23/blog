+++
title = "Résumé"
description = 'My professional résumé'
date = 2026-04-15
draft = false

[menu.main]
  name = "Résumé"
  weight = 10
+++
Please view my résumé embedded below.

<!-- <iframe src="/docs/resume.pdf" width="150%" height="1020px" style="border:none;"></iframe> -->

<style>
  .resume-frame {
    width: 60vw;
    margin-left: calc(50% - 30vw);
  }

  .resume-frame iframe {
    width: 100%;
    height: 1020px;
    border: none;
  }

  @media (max-width: 768px) {
    .resume-frame {
      width: 100vw;
      margin-left: calc(50% - 50vw);
    }
  }
</style>

<div class="resume-frame">
  <iframe src="/docs/resume.pdf"></iframe>
</div>

Or download it directly: [Download PDF](/docs/resume.pdf)
