---
title: "Practice 1: Biological data"
date: 2025-07-01
categories: [practices]
layout: single
author_profile: false
---
# Practice 1

<div class="code-block">
  <button class="copy-button" onclick="copyCode(this)">Copy</button>
  <pre><code class="language-r">
# Primer ejemplo en R
x <- c(1, 2, 3)
mean(x)
  </code></pre>
</div>

<div class="code-block">
  <button class="copy-button" onclick="copyCode(this)">Copy</button>
  <pre><code class="language-r">
# Segundo ejemplo en R
y <- c(10, 20, 30)
sum(y)
  </code></pre>
</div>

<script>
function copyCode(button) {
  const code = button.nextElementSibling.innerText.trim();
  navigator.clipboard.writeText(code).then(() => {
    button.innerText = 'Copied!';
    setTimeout(() => { button.innerText = 'Copy'; }, 1500);
  });
}
</script>

<style>
.code-block {
  position: relative;
  margin: 1em 0;
}
.copy-button {
  position: absolute;
  top: 5px;
  right: 5px;
  background: #4CAF50;
  border: none;
  color: white;
  padding: 5px 8px;
  font-size: 12px;
  cursor: pointer;
  border-radius: 4px;
}
pre {
  background: #f4f4f4;
  padding: 1em;
  border-radius: 6px;
  overflow-x: auto;
  font-family: monospace;
  margin-top: 2em;
}
</style>
