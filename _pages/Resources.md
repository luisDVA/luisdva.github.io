---
layout: single
classes: wide
title: Resources
permalink: /resources/
published: true
share: false
---

# Basic R example

<div class="code-block">
  <button class="copy-button" onclick="copyCode(this)">Copiar</button>
  <pre><code class="language-r"># Código R de ejemplo
x <- c(1, 2, 3, 4, 5)
mean(x)
</code></pre>
</div>

<script>
function copyCode(button) {
  const code = button.nextElementSibling.innerText;
  navigator.clipboard.writeText(code).then(function () {
    button.innerText = "¡Copiado!";
    setTimeout(() => { button.innerText = "Copiar"; }, 2000);
  });
}
</script>

<style>
.code-block {
  position: relative;
  margin: 1.5em 0;
}

.copy-button {
  position: absolute;
  top: 5px;
  right: 5px;
  background: #eee;
  border: 1px solid #ccc;
  padding: 5px 10px;
  font-size: 12px;
  cursor: pointer;
  border-radius: 4px;
}

pre {
  background: #f8f8f8;
  padding: 1em;
  border-radius: 6px;
  overflow-x: auto;
  font-family: monospace;
}
</style>
