# Theory & Interactive Visualizations

## Overview

SP-FM (Shortest-Path Flow Matching) is a conditional flow-matching framework that improves out-of-distribution (OOD) generalisation by **conditioning both the base distribution and the flow field** on the input descriptor. Standard conditional flow matching fixes the base distribution to a single Gaussian and conditions only the velocity field on the descriptor, effectively learning independent flows for each condition. SP-FM replaces this fixed base with a learnable, condition-dependent Gaussian mixture, trained jointly with a shortest-path velocity field.

> **Paper:** [Shortest-Path Flow Matching with Mixture-Conditioned Bases for OOD Generalization to Unseen Conditions](https://arxiv.org/abs/2601.11827) — Rubbi, Akbarnejad, Sanian, Yazdan Parast, Asadollahzadeh, Amani, Akhtar, Cooper, Bassett, Paavolainen, Vakili, Liò, Lotfollahi (2026)

---

## The problem with fixed bases

Standard conditional flow matching samples every condition from the same fixed Gaussian $\mathcal{N}(0, I)$. The velocity field alone must learn the entire mapping from this single base to each target distribution. When a novel condition arrives at test time, there is no mechanism to leverage structural similarities with training conditions — the model can only hope that the conditioned decoder generalises.

This design implicitly treats each condition as an independent estimation problem. The consequences are predictable: performance degrades sharply on unseen conditions, and the degradation worsens as the data dimensionality $D$ grows.

---

## The SP-FM insight

SP-FM replaces the fixed base with a **condition-dependent Gaussian mixture**. The key observation is simple: when two conditions are similar (two drugs with related chemical structures, two genetic perturbations affecting the same pathway), their target distributions are also likely to be similar. By learning a base distribution that reflects this similarity structure, the flow only needs to correct for residual differences rather than learning the entire transport from scratch.

The architecture has two learnable components:

- **$h_\Theta(y)$**: an MLP that predicts GMM mode positions from the condition descriptor
- **$h_p(y)$**: an MLP that predicts mixture weights, passed through a Gumbel-Softmax for differentiable mode selection

At test time, given an unseen descriptor $y_{\text{test}}$, SP-FM predicts a base distribution $\mu_{\text{GMM}}(h_\Theta(y_{\text{test}}), h_p(y_{\text{test}}))$ already positioned near the expected target, then transports it via the learned velocity field.

### Training objectives

SP-FM is trained with two complementary losses:

**Objective 1 — OT flow matching:** Align the learned velocity field $v_\theta$ with the optimal transport path between the target $\rho_n$ and its projected base:

$$\mathcal{L}_{\text{OT}} = \mathbb{E}\left[\sum_{s=1}^{S} \left\| v\left(t, (1-t)x_s^{(0)} + tx_{\tau(s)}, y_n\right) - (x_{\tau(s)} - x_s^{(0)}) \right\|^2 \right]$$

**Objective 2 — Geodesic length:** Encourage the predicted base to remain close to the target by minimising geodesic length on the Wasserstein manifold:

$$\mathcal{L}_{\text{geo}} = \mathbb{E}\left[\sum_{s=1}^{S} \left\| x_{\tau(s)} - x_s^{(0)} \right\|^2 \right]$$

Training alternates between updating the flow model and the base distribution through a scheduled planner with warm-up, alternating, and cool-down phases.

### Explore the architecture

The interactive 3D visualisation below shows how SP-FM's MLP components are wired together — from the condition descriptor input through the base distribution prediction and velocity field to the final generated samples.

<div style="position: relative; width: 100%; height: 0; padding-bottom: 75%; overflow: hidden;">
  <iframe
    src="../spfm_mlp_wired_3d_v2.html"
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: none;"
    allowfullscreen>
  </iframe>
</div>

!!! tip
    Use your mouse to rotate, zoom, and interact with the 3D visualization.

---

## Degrees of freedom analysis

The theoretical justification for SP-FM comes from analysing the **degrees of freedom** in the transport problem on the Wasserstein manifold.

Assume the target distribution $\rho$ is a GMM with $J$ components and fixed mode locations $\Gamma \in \mathbb{R}^{J \times D}$. The base distribution is a GMM with $I$ modes. At test time, the flow $V^* \in \mathbb{R}^{I \times J}$ must be identified from the predicted base parameters. The key result:

!!! abstract "Statement 1 (from the paper)"
    The velocity field $V$ is identified up to $J - ID$ degrees of freedom. If $I \geq \lceil J/D \rceil$, then $V$ is uniquely identified.

The three constraint sources that pin down the transport:

| Source | Count | Origin |
|--------|-------|--------|
| Barycentre conditions | $ID$ | Each mode position $\Theta_i^*$ is the weighted average of target modes it transports to |
| Optimality conditions | $I$ | Equal transport cost across all active modes |
| **Total constraints** | **$ID + I$** | |
| **Variables** | **$I + J$** | Non-zero entries of $V^*$ |
| **Degrees of freedom** | **$J - ID$** | What remains underdetermined |

The formula $\text{DoF} = J - ID$ reveals two critical properties:

**1. Each additional mode reduces DoF by $D$.** Adding one GMM component eliminates $D$ degrees of freedom from the transport problem. This is why multi-modal bases outperform unimodal ones — it is not just about expressiveness, it is about identifiability.

**2. Higher dimensions help SP-FM.** When $D$ is large, each mode contributes proportionally more constraints. This explains the empirical observation that SP-FM's advantage over vanilla CFM becomes *more* pronounced in high dimensions — the opposite of what one might expect.

### The I = 1 collapse

When $I = 1$ (vanilla CFM), the dual optimal transport problem becomes ill-defined. The single dual variable $z_1$ cancels out of the objective entirely:

$$\sup_{z} z_1 - z_1 \sum_{j=1}^{J} q_j + \sum_{j=1}^{J} q_j \|\Theta_1 - \Gamma_j\|^2 = \text{const}$$

The base provides no information about the transport structure. The model must predict all $J$ mixture weights directly from the descriptor — a problem whose generalisation error scales with $J$.

### Interactive exploration

The visualisation below lets you explore these relationships. The left panel shows the constraint budget as you vary $I$, $D$, and $J$. The right panel shows the Wasserstein manifold geometry: the purple dot is the fixed Gaussian base (long geodesic to target), the orange dots are GMM modes (short geodesics). As you increase $I$ or $D$, watch the modes tighten around the target and the degrees of freedom drop to zero.

<div class="spfm-widget" markdown>

<!-- BEGIN INTERACTIVE WIDGET -->
<style>
*{box-sizing:border-box;margin:0;padding:0}
.w-row{display:flex;gap:0;width:100%;border:0.5px solid #ddd;border-radius:12px;overflow:hidden;background:#fafafa}
@media(prefers-color-scheme:dark){.w-row{border-color:#333;background:#151515}}
.w-left{width:220px;min-width:220px;padding:8px 12px;border-right:0.5px solid #ddd;display:flex;flex-direction:column;gap:6px}
@media(prefers-color-scheme:dark){.w-left{border-color:#333}}
.w-right{flex:1;min-width:0;position:relative}
canvas.w-gl{width:100%;height:520px;display:block}
.w-sl{display:flex;align-items:center;gap:6px}
.w-sl label{font-size:11px;color:#888;min-width:28px;font-family:system-ui,sans-serif}
.w-sl span{font-size:12px;font-weight:500;min-width:28px;text-align:right;font-variant-numeric:tabular-nums}
@media(prefers-color-scheme:dark){.w-sl span{color:#ddd}}
@media(prefers-color-scheme:light){.w-sl span{color:#333}}
.w-cards{display:grid;grid-template-columns:1fr 1fr 1fr;gap:4px}
.w-card{border-radius:8px;padding:6px 8px}
@media(prefers-color-scheme:dark){.w-card{background:#1e1e1e}}
@media(prefers-color-scheme:light){.w-card{background:#f0f0f0}}
.w-card .wl{font-size:9px;color:#999;font-family:system-ui,sans-serif}
.w-card .we{font-size:11px;font-weight:500;font-family:system-ui,sans-serif}
@media(prefers-color-scheme:dark){.w-card .we{color:#ccc}}
@media(prefers-color-scheme:light){.w-card .we{color:#444}}
.w-card .wv{font-size:14px;font-weight:500;font-family:system-ui,sans-serif}
.w-bar{position:relative;height:120px;margin:4px 0 0}
.w-leg{display:flex;gap:8px;font-size:10px;color:#999;margin:4px 0 0;flex-wrap:wrap;font-family:system-ui,sans-serif}
.w-dot{width:8px;height:8px;border-radius:2px;display:inline-block;vertical-align:middle;margin-right:3px}
</style>

<div class="w-row">
<div class="w-left">
  <div class="w-sl">
    <label>I</label>
    <input type="range" id="wsi" min="1" max="12" value="4" step="1" style="flex:1" oninput="wupd()">
    <span id="wvi">4</span>
  </div>
  <div class="w-sl">
    <label>D</label>
    <input type="range" id="wsd" min="2" max="500" value="50" step="1" style="flex:1" oninput="wupd()">
    <span id="wvd">50</span>
  </div>
  <div class="w-sl">
    <label>J</label>
    <input type="range" id="wsj" min="4" max="200" value="50" step="1" style="flex:1" oninput="wupd()">
    <span id="wvj">50</span>
  </div>
  <div class="w-cards">
    <div class="w-card"><div class="wl">Variables</div><div class="we">I+J</div><div class="wv" id="wcv" style="color:#666">54</div></div>
    <div class="w-card"><div class="wl">Constraints</div><div class="we">ID+I</div><div class="wv" id="wcc" style="color:#0F6E56">204</div></div>
    <div class="w-card"><div class="wl">DoF</div><div class="we">J−ID</div><div class="wv" id="wcd" style="color:#D85A30">0</div></div>
  </div>
  <div class="w-bar"><canvas id="wbar"></canvas></div>
  <div class="w-leg">
    <span><span class="w-dot" style="background:#1D9E75"></span>Constrained</span>
    <span><span class="w-dot" style="background:#D85A30"></span>Free (DoF)</span>
  </div>
</div>
<div class="w-right">
  <canvas class="w-gl" id="wgl"></canvas>
</div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js"></script>
<script>
(function(){
const dk=matchMedia('(prefers-color-scheme:dark)').matches;
const tH='#1D9E75',cH='#D85A30',pH='#7F77DD';

const bCtx=document.getElementById('wbar');
const bC=new Chart(bCtx,{type:'bar',data:{labels:[],datasets:[
{label:'C',data:[],backgroundColor:tH+'77',borderColor:tH,borderWidth:0.5,borderRadius:2},
{label:'F',data:[],backgroundColor:cH+'77',borderColor:cH,borderWidth:0.5,borderRadius:2}
]},options:{responsive:true,maintainAspectRatio:false,indexAxis:'y',
scales:{x:{stacked:true,grid:{color:dk?'rgba(255,255,255,0.05)':'rgba(0,0,0,0.05)'},ticks:{display:false}},
y:{stacked:true,grid:{display:false},ticks:{color:dk?'#999':'#777',font:{size:10}}}},
plugins:{legend:{display:false},tooltip:{enabled:true}}}});

window.wupd=function(){
const I=+document.getElementById('wsi').value,D=+document.getElementById('wsd').value,J=+document.getElementById('wsj').value;
document.getElementById('wvi').textContent=I;document.getElementById('wvd').textContent=D;document.getElementById('wvj').textContent=J;
const vars=I+J,cons=I*D+I,dof=Math.max(0,J-I*D);
document.getElementById('wcv').textContent=vars;document.getElementById('wcc').textContent=cons;
document.getElementById('wcd').textContent=dof===0?'0':''+dof;document.getElementById('wcd').style.color=dof===0?tH:cH;
const la=[],cD=[],fD=[];
for(let i=1;i<=Math.min(I,8);i++){la.push('I='+i);const c=Math.min(i*D+i,J);cD.push(c);fD.push(Math.max(0,J-c));}
bC.data.labels=la;bC.data.datasets[0].data=cD;bC.data.datasets[1].data=fD;bC.update('none');
wManifold(I,D,J);};

const cv=document.getElementById('wgl'),rr=cv.getBoundingClientRect();
const ren=new THREE.WebGLRenderer({canvas:cv,antialias:true,alpha:true});
ren.setPixelRatio(window.devicePixelRatio);ren.setSize(rr.width,520);
ren.setClearColor(dk?0x151515:0xfafafa,1);
const sc=new THREE.Scene(),cam=new THREE.PerspectiveCamera(40,rr.width/520,0.1,100);
cam.position.set(5,4,7);cam.lookAt(0,0,0);
const ct=new THREE.OrbitControls(cam,cv);ct.enableDamping=true;ct.dampingFactor=0.08;ct.minDistance=3;ct.maxDistance=20;

const mR=40,mV=new Float32Array((mR+1)*(mR+1)*3),mI=[];
for(let i=0;i<=mR;i++)for(let j=0;j<=mR;j++){const u=(i/mR-0.5)*6,v=(j/mR-0.5)*6,idx=(i*(mR+1)+j)*3,r=Math.sqrt(u*u+v*v);
mV[idx]=u;mV[idx+1]=-0.15*r*r+0.02*Math.sin(r*2)*r;mV[idx+2]=v;}
const mG=new THREE.BufferGeometry();mG.setAttribute('position',new THREE.BufferAttribute(mV,3));
for(let i=0;i<mR;i++)for(let j=0;j<mR;j++){const a=i*(mR+1)+j,b=a+1,c=(i+1)*(mR+1)+j,d=c+1;mI.push(a,b,c,b,d,c);}
mG.setIndex(mI);mG.computeVertexNormals();
sc.add(new THREE.Mesh(mG,new THREE.MeshBasicMaterial({color:dk?0x222233:0xdde0ea,transparent:true,opacity:0.25,side:THREE.DoubleSide})));
sc.add(new THREE.Mesh(mG,new THREE.MeshBasicMaterial({color:dk?0x333355:0xbbc0d0,transparent:true,opacity:0.08,wireframe:true,side:THREE.DoubleSide})));

function mY(x,z){const r=Math.sqrt(x*x+z*z);return -0.15*r*r+0.02*Math.sin(r*2)*r;}
const tP=new THREE.Vector3(2.2,0,1.8);tP.y=mY(tP.x,tP.z)+0.05;

function srand(s){return function(){s|=0;s=s+0x6D2B79F5|0;var t=Math.imul(s^s>>>15,1|s);t^=t+Math.imul(t^t>>>7,61|t);return((t^t>>>14)>>>0)/4294967296;}}
function mkrn(seed){let sr=srand(seed);return function(){let u=0,v=0;while(!u)u=sr();while(!v)v=sr();return Math.sqrt(-2*Math.log(u))*Math.cos(2*Math.PI*v);};}
let srg=mkrn(42);const tN=60,tA=new Float32Array(tN*3);
for(let i=0;i<tN;i++){const x=tP.x+srg()*0.25,z=tP.z+srg()*0.25;tA[i*3]=x;tA[i*3+1]=mY(x,z)+0.08;tA[i*3+2]=z;}
const tG=new THREE.BufferGeometry();tG.setAttribute('position',new THREE.BufferAttribute(tA,3));
sc.add(new THREE.Points(tG,new THREE.PointsMaterial({size:0.06,color:dk?0x85B7EB:0x185FA5,transparent:true,opacity:0.5})));
function mkL(t,c){const cv=document.createElement('canvas');cv.width=128;cv.height=32;const x=cv.getContext('2d');x.font='bold 13px sans-serif';x.fillStyle=c;x.textAlign='center';x.fillText(t,64,20);return new THREE.Sprite(new THREE.SpriteMaterial({map:new THREE.CanvasTexture(cv),transparent:true,opacity:0.7}));}
const tL=mkL('target',dk?'#85B7EB':'#185FA5');tL.position.set(tP.x,tP.y+0.5,tP.z);tL.scale.set(1,0.3,1);sc.add(tL);

const vG=new THREE.Group(),sG=new THREE.Group(),gG=new THREE.Group(),dG=new THREE.Group();
sc.add(vG);sc.add(sG);sc.add(gG);sc.add(dG);
function clr(g){while(g.children.length)g.remove(g.children[0]);}
function mkC(p1,p2,col,op,lf){const m=new THREE.Vector3().addVectors(p1,p2).multiplyScalar(0.5);m.y+=lf||0.5;
const c=new THREE.QuadraticBezierCurve3(p1,m,p2),pts=c.getPoints(30);
return new THREE.Line(new THREE.BufferGeometry().setFromPoints(pts),new THREE.LineBasicMaterial({color:col,transparent:true,opacity:op}));}

window.wManifold=function(I,D,J){
clr(vG);clr(sG);clr(gG);clr(dG);
const vP=new THREE.Vector3(0,mY(0,0)+0.05,0);
const vs=new THREE.SphereGeometry(0.08,12,12);
vG.add(new THREE.Mesh(vs,new THREE.MeshBasicMaterial({color:new THREE.Color(pH),transparent:true,opacity:0.8})).translateX(vP.x).translateY(vP.y).translateZ(vP.z));
vG.add(mkC(vP,tP,new THREE.Color(pH),0.4,0.8));
const vL=mkL('N(0,I)',pH);vL.position.set(vP.x,vP.y+0.45,vP.z);vL.scale.set(0.9,0.28,1);vG.add(vL);

const dof=Math.max(0,J-I*D),mI2=Math.min(I,8),aS=Math.PI*2/Math.max(mI2,3),bR=0.6+mI2*0.12,sP=[];
for(let i=0;i<mI2;i++){
const a=aS*i-Math.PI/2,bx=tP.x+Math.cos(a)*bR*(0.8+dof/Math.max(J,1)*1.5),bz=tP.z+Math.sin(a)*bR*(0.8+dof/Math.max(J,1)*1.5),by=mY(bx,bz)+0.05;
sP.push(new THREE.Vector3(bx,by,bz));
const d=new THREE.Mesh(new THREE.SphereGeometry(0.06,8,8),new THREE.MeshBasicMaterial({color:new THREE.Color(cH),transparent:true,opacity:0.9}));
d.position.copy(sP[i]);dG.add(d);
gG.add(mkC(sP[i],tP,new THREE.Color(cH),0.5,0.3));}
if(mI2>=3){const rP=[];for(let i=0;i<=mI2;i++)rP.push(sP[i%mI2]);
sG.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(rP),new THREE.LineBasicMaterial({color:new THREE.Color(cH),transparent:true,opacity:0.2})));}
const gL=mkL('GMM(I='+I+')',cH),ax=sP.reduce((s,p)=>s+p.x,0)/sP.length,az=sP.reduce((s,p)=>s+p.z,0)/sP.length;
gL.position.set(ax,mY(ax,az)+0.7,az);gL.scale.set(1.1,0.3,1);sG.add(gL);
const dL=mkL('DoF='+dof,dof===0?tH:cH);dL.position.set(ax,mY(ax,az)+1.1,az);dL.scale.set(1,0.28,1);sG.add(dL);};

let aR=true,iT=null;
ct.addEventListener('start',()=>{aR=false;if(iT)clearTimeout(iT);});
ct.addEventListener('end',()=>{iT=setTimeout(()=>{aR=true;},3000);});
(function an(){requestAnimationFrame(an);
if(aR){const r=Math.sqrt(cam.position.x**2+cam.position.z**2),th=Math.atan2(cam.position.x,cam.position.z)+0.0015;
cam.position.x=r*Math.sin(th);cam.position.z=r*Math.cos(th);}
ct.update();ren.render(sc,cam);})();

wupd();
window.addEventListener('resize',()=>{const r=cv.parentElement.getBoundingClientRect();ren.setSize(r.width,520);cam.aspect=r.width/520;cam.updateProjectionMatrix();});
})();
</script>
<!-- END INTERACTIVE WIDGET -->

</div>

---

## Why this matters for high-dimensional biology

Single-cell transcriptomics typically operates in $D = 50$–$500$ dimensions (PCA components of the gene expression space). At these scales, the fixed-base approach is at a severe disadvantage:

- With $D = 50$ and $J = 50$ target components, even $I = 1$ leaves $\text{DoF} = 0$. But this is misleading — the $I=1$ dual is ill-defined regardless (Statement 1, §2 in the paper).
- With $D = 500$, a single additional mode ($I = 2$) eliminates 500 degrees of freedom. The constraint budget is dominated by the $ID$ term, so high-dimensional data makes each mode disproportionately informative.

This is precisely what the ablation experiments confirm (Figure 3b in the paper): CFM's energy distance grows by over an order of magnitude between 50 and 500 PCA components, while SP-FM maintains stable performance across the full range.

---

## Summary

| Property | Vanilla CFM | SP-FM |
|----------|-------------|-------|
| Base distribution | Fixed $\mathcal{N}(0, I)$ | Learned GMM conditioned on $y$ |
| Transport distance | Long (origin to target) | Short (near-target mode to target) |
| DoF at test time | $J - D$ (ill-defined for $I=1$) | $J - ID$ (well-posed for $I \geq \lceil J/D \rceil$) |
| High-$D$ behaviour | Degrades | Improves |
| OOD mechanism | Hope decoder generalises | Base encodes condition similarity |
