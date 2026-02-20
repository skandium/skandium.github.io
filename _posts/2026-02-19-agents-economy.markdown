---
layout: single
classes: narrow
title:  "The macroeconomics of agentic AI"
date:   2026-02-17
categories: general
comments: false
header:
  og_image: /assets/images/agent_economy/race2.svg
---

During the Industrial Revolution, machines displaced most Western agricultural workers, who later went on to cities and earned higher wages. However, some time later, the automobile displaced all horses, and they still haven't reallocated to new professions. In the coming decades, are we the 19th century peasant, or the horse?

## The short term - estimating task automation exposure

To estimate the near term productivity gain effect from AI, we can look at the tasks/professions exposed to AI technologies in the near future, aggregate their share of the overall wage bill, and use Hulten's theorem to approximate the TFP growth:

$$

\text{TFP growth} =  \text{GDP share of tasks impacted by AI} \times \text{Average labour cost savings} = \\ \text{Tasks exposed to AI} \times \text{Automatable within 10 years}\times \text{Average cost saving} \times \text{Labour share of production}

$$

Several authors have done exactly this, with the following estimates:


| Variable                     | Acemoglu (2024) [[2]](#ref-2) | Aghion & Bunel (2024) [[3]](#ref-3)  | OECD (2024) [[4]](#ref-4) |
|-----------------------------|-------|---| --- |
| Tasks exposed to AI   | 20%   | 18.5 - 68% | 12% - 50% | 
| Automatable within 10 years | 23%   |  23 - 80%  |  23% - 40%  |
| Average cost saving from AI | 27%   |  27 - 40%  |  30% |
| Labour share of production  | 53.5% |  57% |  -  |
| ---- | ---- | ---- | --- |
| Annual Total Factor Productivity growth | 0.07pp | 0.07 - 1.24pp |  0.25 - 0.6pp  |

Looking back from 2026, when Claude Code promises to decrease the price of building software, and Anthropic and OpenAI are aiming to revolutionise all of computer usage, we can nitpick at some of the underlying estimates. For example, that only 23% of exposed tasks can be automated profitably within 10 years is based on a computer vision study from 2024. The average cost saving of 27% is based on pilot studies that used rudimentary models - GPT-3 and ChatGPT-3.5. The 20% is based on exposure estimates from the famous GPTs are GPTs paper [[1]](#ref-1), which was published when GPT-4 was the best model. However, I think the main limitation to increased exposure is the lack of embodied AI. As defined by Acemoglu, an occupation is exposed if 50% of its tasks are AI-exposed. There are simply large areas of the economy (manufacturing, hospitality, food services etc) where digital assistants can't help much.

Anyway, the wide range of input estimates between authors highlights the uncertainty of these predictions, so here's a calculator to project your own:

<div style="background: #1e222a; padding: 20px; border-radius: 8px; margin: 20px 0; border: 1px solid #3a3f4b;">
<h4 style="margin-top: 0;">TFP Growth Calculator</h4>
<div style="display: grid; gap: 12px; margin-bottom: 16px;">
  <label>Timeframe: <strong id="v5">10</strong> years
    <input type="range" id="timeframe" min="1" max="10" value="10" style="width: 100%;" oninput="calcTFP()">
  </label>
  <label>Tasks exposed to AI: <strong id="v1">20</strong>%
    <input type="range" id="exposed" min="5" max="100" value="20" style="width: 100%;" oninput="calcTFP()">
  </label>
  <label>Automatable within 10 years: <strong id="v2">23</strong>%
    <input type="range" id="automatable" min="10" max="100" value="23" style="width: 100%;" oninput="calcTFP()">
  </label>
  <label>Average cost saving: <strong id="v3">27</strong>%
    <input type="range" id="saving" min="10" max="100" value="27" style="width: 100%;" oninput="calcTFP()">
  </label>
  <label>Labour share of production: <strong id="v4">54</strong>%
    <input type="range" id="labour" min="40" max="100" value="54" style="width: 100%;" oninput="calcTFP()">
  </label>
</div>
<div style="font-size: 1.1em;">
  <strong>Annual TFP growth: <span id="result">0.07</span>pp</strong>
</div>
<div id="comparison" style="margin-top: 12px; padding: 10px; border-radius: 4px; background: #2d323c;">
</div>
</div>

<script>
function calcTFP() {
  var exposed = document.getElementById('exposed').value / 100;
  var automatable = document.getElementById('automatable').value / 100;
  var saving = document.getElementById('saving').value / 100;
  var labour = document.getElementById('labour').value / 100;
  var timeframe = parseInt(document.getElementById('timeframe').value);

  document.getElementById('v1').textContent = Math.round(exposed * 100);
  document.getElementById('v2').textContent = Math.round(automatable * 100);
  document.getElementById('v3').textContent = Math.round(saving * 100);
  document.getElementById('v4').textContent = Math.round(labour * 100);
  document.getElementById('v5').textContent = timeframe;

  var totalGrowth = exposed * automatable * saving * labour;
  var tfp = (Math.pow(1 + totalGrowth, 1 / timeframe) - 1) * 100;
  document.getElementById('result').textContent = tfp.toFixed(2);

  var comp = document.getElementById('comparison');
  var elec = 1.3, ict = 1.0;
  var pctElec = (tfp / elec * 100).toFixed(0);
  var pctIct = (tfp / ict * 100).toFixed(0);

  comp.innerHTML = '<strong>Historical comparison:</strong><br>' +
    'Electrification (1920s) (1.3pp/yr): ' + pctElec + '% of impact<br>' +
    'ICT revolution (1995-2005) (1.0pp/yr): ' + pctIct + '% of impact';

  if (tfp >= 1.0) {
    comp.style.background = '#1b4332';
  } else if (tfp >= 0.5) {
    comp.style.background = '#3d3522';
  } else {
    comp.style.background = '#2d323c';
  }
}
calcTFP();
</script>

## The long term - an economic model

While Hulten's decomposition provides a useful first order approximation to immediate effects of AI automation, we need a full model to understand the long term comparative statics of output, wages and labour share. The model in Acemoglu [[2]](#ref-2) is reasonable starting point for static analysis, as it allows heterogenous tasks with varying degrees of automatability. Assuming fixed stocks of capital and labour, he models the economy as producing output from a continuum of tasks indexed by $z \in [0,N]$:

$$
Y
=
\left(
\int_0^N \big(B\,y(z)\big)^{\frac{\sigma-1}{\sigma}} dz
\right)^{\frac{\sigma}{\sigma-1}}
$$

where:

- $Y$ is total output  
- $y(z)$ is output of task $z$  
- $B$ is overall productivity  
- $\sigma$ is the elasticity of substitution across tasks, $> 1$.
- $N$ is the total number of tasks in the economy  

Tasks differ in how efficiently they can be performed by labour versus capital. Each task can be produced using either labour or capital:

$$
y(z)
=
\begin{cases}
A_K \gamma_K(z) k(z), & z \le I \quad \text{(automated tasks)} \\
A_L \gamma_L(z) l(z), & z > I \quad \text{(labour tasks)}
\end{cases}
$$

where:

- $I$ is the automation frontier — the highest task index that capital can perform  
- $A_L, A_K$ are labour- and capital-augmenting productivity terms  
- $\gamma_L(z), \gamma_K(z)$ describe how suitable each task is for labour or capital  
- $L$ is total labour supply  
- $K$ is total capital stock  

Firms assign each task to whichever factor can perform it at lower cost. Tasks below the automation frontier are performed by capital, while the remaining tasks are performed by labour.

---

Solving the model for wages gives:

$$
w
=
\left(\frac{Y}{L}\right)^{\frac{1}{\sigma}}
(B A_L)^{\frac{\sigma-1}{\sigma}}
\left(
\int_I^N \gamma_L(z)^{\sigma-1} dz
\right)^{\frac{1}{\sigma}}
$$

AI automation affects this equation through three channels:

1. Displacement effect - Automation increases $I$, reducing the number of tasks performed by labour. This reduces labour demand and lowers wages.
2. Productivity effect - Automation reduces production costs, increasing overall productivity $\frac{Y}{L}$. This raises wages.
3. New task creation - AI may create entirely new tasks, increasing $N$. If these tasks are performed by labour, they increase labour demand and wages.

The overall effect of AI on wages depends on the balance between these effects. Empirically, displacement effects have dominated in sectors such as manufacturing, contributing to a declining labour share and a downward pressure on wages. [[6]](#ref-6)[[7]](#ref-7) However, there are counterexamples, such as the introduction of ATMs not reducing teller jobs. [[9]](#ref-9)

This toy model is extended in [[5]](#ref-5). While the treatment is too complex to mention here, the key insights remain similar: 

1. In the short run, unlike factor-augmenting technologies (human multipliers), automating more tasks always reduces labour demand and can decrease wages.
2. In the long run, capital adjusts. Due to this, an automation increase can **increase wages**. In fact, it's precisely when AI is extremely more efficient in the automated tasks that wages grow most. Intuitively, if a task is going to be automated anyway, it is better if it results in large productivity gains, as this transfers to wages, the scarce production factor.
3. In the long run, an automation ($\uparrow I$) increase still reduces employment.
4. The creation of new tasks ($\uparrow N$) counteracts this and increases wages and employment.
5. Therefore, the race between man and machine, or between new task creation $N$ and the automation frontier $I$ determines the long term outcome of employment.

![race](/assets/images/agent_economy/race2.png)

In the context of agentic AI, this implies that if it's mostly a factor-augmenting technology - a 100x engineer pathway, where labour is needed to actively orchestrate tens of agents - wages increase and there is strong labour demand. However, if the agents are able to set and execute (sub)tasks autonomously, with supervision input not providing much value, then we are on the automation increase pathway. This can mean long run wage improvement, but will also decrease labour demand, unless new labour-specific tasks can be generated.

### A path to AGI?

There are some authors, notably Erik Brynjolfsson and Anton Korinek, that take a far more techno-optimistic view, and even chart a theoretical path towards AGI. Tremmel [[8]](#ref-8) summarizes these approaches on the economics of *transformative AI*. The underlying assumption here is that $I \to N$, meaning that all tasks in the economy become automatable, or that AGI will be reached. This then implies a production function $Y = A_K K + A_L L$, meaning that labour and capital are perfect substitutes (across all tasks). Even without a notable increase in capital productivity $A_K$, the fact that capital can accumulate, drives exponential growth and the share of labour falls to zero. This is the economic version of self-replicating machines: once capital can build more capital, growth feeds on itself. Regarding the possibility of new task creation, Tremmel hints toward a fundamental constraint: human abilities are not unlimited, and once these are replicated by robots, it's not possible to create new labour tasks. While these strong assumptions are interesting for exploring the models in the limit, it's more related to science fiction than today's technology.

A more plausible argument is for the automation of research. On a high level, the economy can be modelled as consisting of two activities: production and R&D. The latter, including both scientific research but also designing new products, is what enables productivity improvements over the long term. The models including research are called _endogenous_, as in, they do not take technological improvements as exogenous manna from heaven. An important distinction with production improvements we looked at before is that improvements to technological development can accrue in perpetuity. Endogenous models often relate long term growth rates to the population size in one way or another, since this also affects the number of scientists. The more people around, the higher the chance to produce an Einstein. Given that population growth rates are tanking all across the world, a labour augmenting or substituting automated researcher would clearly improve growth. An oft quoted achievement is the Nobel Prize in Chemistry awarded to Demis Hassabis (AlphaFold).

### My own take

I remain bullish on both AI and labour. Having worked with neural networks for 10 years (including building customer support agents to automate humans, and now using Claude Code daily), I think that while they are the closest thing to AI we have managed to create, they have serious fundamental generalization and efficiency problems compared to our minds. This might not be surprising - I've always found the parallels between neuroscience and ML models to be somewhat contrived. After all, AI is *artificial*, meaning  that while it can approximate humans in economic efficiency in some tasks, it retains its unique, synthetic internal logic. However, this marked difference from humans is exactly what provides task-specific comparative advantages to labour. I don't believe in the automation of research in the near future, as the frontiers of knowledge creation are exactly where LLMs struggle.

Today's tools such as Claude Code, are markedly amazing, and even without underlying improvements in the LLMs, will become even better just because of better scaffolding and organisations adapting to their use. However, they remain a strictly labour-augmenting technology, meaning that their quality in fully autonomous mode is poor. I do acknowledge that some metrics such as METR's [AI ability over long tasks](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/) show exponential gains, which makes predicting the future difficult. Intuitively, this means that at least domains providing a fast feedback loop with exact verification (of which programming is the best example) will be significantly disrupted. Whether this is sufficient to fully automate software engineering jobs (of which programming is only a subset of tasks) remains to be seen. However, it's difficult to see how full automation could be achieved in other sectors.

Despite the hyperbole seen in some accelerationist circles, the best near term predictions for TFP growth put the impact of AI at roughly half of the 1995-2005 ICT boom in the optimistic scenario, while the pessimistic estimates are near zero.

### References

<span id="ref-1">[1]</span> [GPTs are GPTs: An Early Look at the Labor Market Impact Potential of Large Language Models](https://arxiv.org/pdf/2303.10130) <br>
<span id="ref-2">[2]</span> [The Simple Macroeconomics of AI](https://shapingwork.mit.edu/wp-content/uploads/2024/05/Acemoglu_Macroeconomics-of-AI_May-2024.pdf#:~:text=The%20nexus%20of%20the%20many,the%20capital%20stock%20of%20the) <br>
<span id="ref-3">[3]</span> [AI and Growth: Where Do We Stand?](https://www.frbsf.org/wp-content/uploads/AI-and-Growth-Aghion-Bunel.pdf) <br>
<span id="ref-4">[4]</span> [Miracle or Myth? Assessing the macroeconomic productivity gains from Artificial Intelligence](https://www.oecd.org/content/dam/oecd/en/publications/reports/2024/11/miracle-or-myth-assessing-the-macroeconomic-productivity-gains-from-artificial-intelligence_fde2a597/b524a072-en.pdf) <br>
<span id="ref-5">[5]</span> [The Race between Man and Machine: Implications of Technology for Growth, Factor Shares, and Employment](https://economics.mit.edu/sites/default/files/publications/The%20Race%20Between%20Man%20and%20Machine%20-%20Implications%20of.pdf) <br>
<span id="ref-6">[6]</span> [Robots and Jobs: Evidence from US Labor Markets](https://shapingwork.mit.edu/wp-content/uploads/2023/10/Robots-and-Jobs-Evidence-from-US-Labor-Markets.p.pdf) <br>
<span id="ref-7">[7]</span> [Automation and New Tasks: How Technology Displaces and Reinstates Labor](https://www.aeaweb.org/articles?id=10.1257/jep.33.2.3) <br>
<span id="ref-8">[8]</span> [Economic Growth under Transformative AI](https://philiptrammell.com/static/economic_growth_under_transformative_ai.pdf) <br>
<span id="ref-9">[9]</span> [Toil and Technology](https://www.imf.org/external/pubs/ft/fandd/2015/03/bessen.htm)