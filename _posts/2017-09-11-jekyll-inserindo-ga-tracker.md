---
layout: post
title: Como ativar o Google Analytics em uma Github Page 
category: Jekyll
tags: [jekyll, github-pages, google-analytics]
---

Acabei de ativar o tracker do Google Analytics aqui nesse site. É uma boa hora para explicar como fiz.

Aqui eu presumo que você já tem a sua Github Page com Jekyll e já sabe utilizar o Analytics.

O primeiro passo foi criar uma propriedade do site dentro da minha conta no Analytics.

Após isso, o Analytics fornece um Tracking Code único para identificar o site assim como o código JavaScript para
ativar o tracking.

Uma boa prática é guardar o Tracking Code em uma variável, assim podemos facilmente utilizá-la em diversos lugares. 

Vamos criar a variável inserindo o conteúdo no arquivo "_config.yml":

**_config.yml:**

<pre><code class="code yaml">ga_tracking_id: "UA-XXXXXXXX-X"</code></pre>

Agora, sempre que quiser utilizar a variável, vamos usar <code class="code twig">{% raw %}{{ site.ga_tracking_id }}{% endraw %}</code>.

Abaixo está o código HTML/JavaScript para ativar o tracking e o melhor é armazená-lo em um arquivo separado, no diretório
"_includes" também para reusá-lo sempre que necessário.

**_includes/analytics.html (remover os espaços das tags "script")**:

<pre><code class="code html">< script >
    (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
            (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
        m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
    })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

    ga('create', '{{ site.ga_tracking_id }}', 'auto');
    ga('send', 'pageview');
</ script >
</code></pre>
 
Agora, sempre que quisermos ativar esse código, basta incluir o arquivo "analytics.html".

No meu site, incluí o código no layout "default.html" que é o layout principal do site.

Uma melhoria que podemos fazer é somente ativar o tracking em ambiente de produção, para não ativá-lo no ambiente de 
desenvolvimento e enviar estatísticas erradas para o Analytics.

Assim ficou o include no **"_layouts/default.html"**:

<pre><code class="code twig">{% raw %}{% if site.ga_tracking_id and jekyll.environment == 'production' %}
    {% include analytics.html %}
{% endif %}
{% endraw %}</code></pre>

Eu fiz essas mudanças nesse commit aqui, acho que fica mais fácil de conferir: 
[Implementa o tracking do Google Analytics em prod](https://github.com/brunohanai/brunohanai.github.io/commit/eaeebb41dd71e72cd937989440af178871bb19bb)