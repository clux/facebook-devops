
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

### Fast & secure config-managed kubernetes continuous delivery

- Eirik Albrigtsen : [github.com/clux](https://github.com/clux)
- Babylon Health : [github.com/Babylonpartners](https://github.com/Babylonpartners)

<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 560px" />

NOTES:
- Hi. Eirik. Platform, this is Buzzword Title.
- "Things to maybe consider when designing a CD system on kube"
- Not prescribing a particlar tool for the job here, just strategies
- Will touch our public tooling and helm.

---
<!-- .slide: data-background-image="/chef.gif" data-background-size="100% auto" class="color"-->

notes:
- so glad on kube => deployments.
- never want to be doing chef surgery again
- not that the tool is necessarily bad, but wrong pattern; handover
- deploy,rs,rupgrade best selling points of kube.
- imply we can deploy as fast as we want.

---
<!-- .slide: data-background-image="/rolling.gif" data-background-size="100% auto" class="color"-->
<img src="./kube.png" style="border: none; opacity: 0.7; width: 120px; position: absolute; top: 60px; right: 120px" />
notes:
- and we've definitely gone all out on this
- context: kube 1y ago, 400 devs, 400 microservices, in ~20 clusters
- managing all this is a complex problem
- what I'm covering might not be relevant unless you req. stupid scale
- we have found some speedbumps and patterns
- anyway; onto serious part of presentation, workflow :shrug:

---
<!-- .slide: data-background-color="#353535" -->
Desired flow

```sh
dev → git app repo x → ci ↘
dev → git app repo y → ci → registry ??? kubernetes
dev → git app repo z → ci ↗
```

notes:
- teams own app repos, all managing own ci (image build, tests)
- ? => anything. webhook from registry hitting a job doing rolling upgrade (mention later)
- where is the yaml? no standards? chart museum, but hard to verify changes to it. hard to recreate if a cluster goes down.
- i cannot believe i am at facebook advocating for monorepos :shrug:
- pause..
- but more specifically, a yaml monorepo

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Definitions?

<ul>
  <li class="fragment">What is a service?</li>
  <li class="fragment">What distinguishes them?</li>
  <li class="fragment">What abstractions do we want?</li>
</li>

notes:
- q1 maths bg => asking "what do you mean by X?"
- kube answer, full shebang comparison, dev shielding
- dev answer,  (image, tag, yaml) - want them to recognise parts of yaml
- q2 ports, resource use, replicas, evars, health
- q3 many different ways of accomplishing similar results
- but less variance gives an easier system to make global change to
- get sensible defaults, needed: no health check, no resources, no sa
- avoid loading complexity on all devs when enforcing

---
<!-- .slide: class="color" style="min-width: 100%" -->
helm charts

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: {{ .Values.name }}
        command:
{{ toYaml .Values.command | indent 8 }}
        resources:
{{ toYaml .Values.resources | indent 10 }}
        ports:
```
```yaml
{{- range $p := .Values.ports }}
        - name: {{ $p.name }}
          containerPort: {{ $p.port }}
          protocol: {{ $p.protocol }}
{{- end }}
```

notes:
- follow ticketmasters one chart to rule them all approach (good talk)
- natural conclusion: a general chart - easier than forking. looks like this ^
- test is trying out all paths for the template (slow/kubeval/diff)
- strange read, these charts. weird tpl fns, indent pattern:why 8 != 10?
- step back: tpl yaml, takes yaml, passed.to.toYaml - then indented..

---

<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
<!-- .slide: data-background-image="/morty-steps.png" data-background-size="100% auto" style="color: #000;" class="color"-->


<div style="position: relative; text-align: left; top: 450px; left: 415px; font-size: 24px">x: <br/>&nbsp;&nbsp;&nbsp;&nbsp;{{ toYaml .Values.X | indent 10 }}</div>


notes:
- really convoluted assignment
- hard to justify one chart approach with multi-stage yaml when just go down one code path anyway?
- what happened to using an API? we could have had static type checks on this!

---
<!-- .slide: data-background-image="/massage.gif" data-background-size="100% auto" style="color: #ff0;" class="color"-->

<div style="position: relative; top: 100px; left: 0px; font-size: 30px; overflow: visible;">{{- include "cont-env" (merge (dict "root" $) .Values.env) | trim | nindent 8 }}</div>

notes:
- but instead we massaging yaml into this shit just to pass two arguments to a fn
- and instead i have to do a cluster wide helm diff when changing a template
- cause that's the implication of a massive chart without types
- and this isn't even the most controversial parts of helm
- breathe...
- but there's hope; helm 3 coming around

---
<!-- .slide: data-background-image="/zapp.gif" data-background-size="100% auto" style="color: #ff0;" class="color"-->

<div style="position: relative; top: 100px; left: 500px; font-size: 30px; overflow: visible;">Lua</div>

notes:
- they're putting lua inside of it!
- this will probably FIX EVERYTHING.
- -> anyway, so i was talking monorepos, right?


---
<!-- .slide: data-background-color="#353535" -->
Desired config management repo

```sh
manifests
 ├── config.yml
 └── services
     ├── blog
     │   └── manifest.yml
     └── webapp
         ├── staging.yml
         ├── prod.yml
         ├── prod-uk.yml
         ├── prod-ca.yml
         └── manifest.yml
```
notes:
- config monorepo => paper trail (medical company)
- sanity checks with req. statuses (b4 admission ctrls were ez)
- quick feedback, easy to catch mistakes, self-service (can self checks)
- want developers to be dealing with a PaaS (no proliferation of kube yaml)
- default manifest, overrideable by environment, and by region
- logging -> dev, autoScaling -> prod
- base defaults (cfgs for region and org) -> less stuff in charts

---
<!-- .slide: data-background-color="#353535" -->
Upgrade flow

```sh
dev → git app repo x → ci ↘
dev → git app repo y → ci → registry → kubernetes
dev → git app repo z → ci ↗
```

```sh
dev ↘
dev → git manifest repo → ci → kubernetes
dev ↗
```

notes:
- two flows for such a system; top flow for staging/rolling environments
- reconciliation loop always active (talk later): versions hardcoded 4 prod
- either case, need a valid manifest for every service regardless
- (flux similar flow diag)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
shipcat 🚢😾

<ul>
  <li class="fragment">Defines `Manifest`</li>
  <li class="fragment">Defines `Config`</li>
  <li class="fragment">Converts them to CRDs from filesystem</li>
  <li class="fragment">Versions our platform</li>
</li>

notes:
- rust tool for standardisation, is open source; but not selling it here
- trying to keep it untied from our platform - but obvious feature parity
- whitelist features for k8s
- tons of validation; typed yaml,  >> templates
- useable abstraction - crds become an api, tests can get canonical info
- versioning: same as rust; shout at you until you get it right

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

shipcat cli

```bash
shipcat values webapp    # <-> helm get values
shipcat template webapp  # <-> helm template {basechart}
shipcat apply webapp     # <-> helm upgrade --install
shipcat validate webapp  # --- helm lint
```


notes:
- leverages helm for templating, and other things (for now)
- acts on a kube context
- CD tool - useable by devs
- validation: reduces criticality of chart (specifics later)

---

<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Resources interface

```yaml
resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 500Mi
```

notes:

- sanity errors (max 32GB per replica - largest kube node)
- req > lim
- extra validation: missing structs / naming errors (deny_unknown_fields)
- typos: request: typos most common error; typos cascade weirdly

---

<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Configs interface

```yaml
configs:
  mount: /config/
  files:
  - name: logging.conf.j2
    dest: logging.conf
  - name: newrelic-python.ini.j2
    dest: newrelic.ini
```

notes:
- abstraction around config files because of common patterns:
  * newrelic configs injected in a lot of pods
  * python logging configuration files
- provides an easy wrapper around all the yaml you need in kube to:
  * template then inline large configs in a ConfigMap
  * use the configmap as a volume
  * volume mount each file in the config map onto the file system (want individually!)
- can check that templates render


---

<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Metadata interface

```yaml
metadata:
  team: Platform
  repo: https://github.com/clux/webapp-rs
```

NOTES:

- not used by kube, accountability, although people do labels for this
- team unique key, team needs slack channels set, github admins teams
- used to generate vault policies, notifications with links to github diffs
- soon rbac policies for namespaces

---
```yaml
name: webapp
image: clux/webapp-rs
version: 0.2.0
env:
  DATABASE_URL: IN_VAULT
resources:
  requests:
    cpu: 50m
    memory: 100Mi
  limits:
    cpu: 200m
    memory: 200Mi
replicaCount: 2
```
```yaml
health:
  uri: /health
httpPort: 8000
regions:
- minikube
metadata:
  team: Platform
  repo: https://github.com/clux/webapp-rs
```
notes:
- complete look
- secret lookup encoded in there as well, kind of simplistic

---
<!-- .slide: data-background-image="/cop.gif" data-background-size="100% auto" style="color: #ff0;" class="color"-->

<div style="position: relative; top: 420px; left: 0px; font-size: 30px; overflow: visible;">helm upgrade --install</div>

notes:
- let's talk about helm upgrades
- releases have FAILED/PENDING/DEPLOYED states

---
<!-- .slide: data-background-image="/science.gif" data-background-size="100% auto" style="color: #000;" class="color"-->

<div style="position: relative; top: 350px; left: -280px; font-size: 30px; overflow: visible;">Error: UPGRADE FAILED: "svc" has no deployed releases</div>

notes:
- bug 3353 jan2018: basically wontfixed
- too hard to diff and recover - even if kube knows how to do this
- but whatever, can deal? first time thing right?
- yeah, cluster recovery bit of a pain, but w/e
- however...

---
<!-- .slide: data-background-image="/guesswork.gif" data-background-size="100% auto" style="color: #2fa;" -->

<div style="position: relative; top: -320px; font-size: 36px">helm upgrade --wait --timeout=300</div>

notes:
* helm upgrade --wait can also proc this: bug 4387 jul2018 open
* bad because similar to initial failure => needs purge.
* History LIMIT <- why: helm ls
* We only needed simple bool for notifies; link to logs + job in channel!

---
<!-- .slide: data-background-image="/what.gif" data-background-size="100% auto" style="color: #2fa;" -->

<div style="position: relative; top: -320px; font-size: 36px">please wait <i>n</i> seconds</div>

NOTES:
* just poll rsets direct, but how long to wait?

---

worst case timeout

```rust
fn estimate_wait_time(&self) -> u32 {
  let iterations = self.estimate_rollout_iterations();
  let delay = self.health.wait; // == initialDelaySeconds
  let size = self.imageSize; // in MB
  let pulltime = max(60, size*90/512); // 90s / 512MB ok ?
  (delay + pulltime) * iterations
}
```

notes:
* only like 3 factors:
* cycles: 8 pods, 4 cycles, but 100% surge => 1 cycle
* initialDelaySecs (probe), kube will wait at least this long
* finger in air estimate for network..
* how to compute; cycles*estimated(pull+upgrade)
* know how to upgrade, wait. polling rs exercise for the reader.

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Reconciliation: take 1

```bash
for s in $(shipcat list-services -r {region}); do
  shipcat values $s -s | helm upgrade basechart $s --install
done
```

notes:
* pseudocode from now on; but actual shipcat commands
* no indication that you actually upgraded anything, helm upgrade returns
* no way to notify about successful configurations (since no change)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Reconciliation: take 2

```bash
for s in $(shipcat list-services -r {region}); do
  shipcat apply $s
done
```

notes:
* apply here is a wrapper around helm diff and helm upgrade
* upgrade iff diff is non-empty
* => can notify on every upgrade we start - success/fail if wait succeeded
* helm diff is slow, need to fetch all secrets
* tiller has internal queue, can't scale
* can parallelize this (ampersand, but pooled - but numjobs scales with tiller)

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Reconciliation: take 3

```bash
for s in $(shipcat list-services -r {region}); do
  res=$(shipcat crd $s | k apply -f -)
  if [[ $res =~ "configured|created" ]]; do
    shipcat apply $s
  fi
done
```

notes:
* one job like this for every region spun off after merging to manifests
* 1 min to apply crds for 300 svcs (limited by wait time)
* avoids helm diff everything - crds are super quick to construct and apply
* don't get auto-secret propagation

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Reconciliation: take 4

```bash
for s in $(shipcat list-services -r {region}); do
  res=$(shipcat crd $s | k apply -f -)
  if [[ $res =~ "configured|created" ]]; do
    shipcat template $s -s | k apply -f -lapp=$s --prune -
  fi
done
```

notes:
* tiller free - can direct to arbitrary namespaces
* crd based ones => two sets => auto-gc
* crd controlled <= owner refs future

---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->
Reconciliation: take operator?

```bash
for s in $(shipcat list-services -r {region}); do
  shipcat crd $s | k apply -f -
done
```

notes:
* write you a tiller
* want to be careful with that because tiller == escalation machine
* even if rbac namespace tiller; helm access => tillers access
* access to special operations?
* logs? storage, history apis, retries?
* currently piggy backing on ci

---

<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

Do YOU need tiller? helm?

<ul>
  <li class="fragment highlight-red">rollbacks</li>
  <li class="fragment highlight-red">grouping resources</li>
  <li class="fragment highlight-red">diffs</li>
  <li class="fragment highlight-red">single source of truth?</li>
  <li class="fragment highlight-green">templating</li>
</li>

notes:
- rollbacks -> unsafe illusion (cant rollback db) -> `git backout`
- grouping resources together -> owner refs || labels.. (`apply --prune -lapp`)
- diffing (helm diff plugin) -> kube diff (1.13)
- gitops style repos with helm => 3 sources of truth (2 gateways for bugs)
- templating -> a totally sane industry standard
- seriously; start with helm templates, just don't use tiller, pipe.

---
<!-- .slide: data-background-image="/applause.gif" data-background-size="100% auto" class="color"-->
---
<!-- .slide: data-background-color="#353535" class="center color" style="text-align: left;" -->

## Thank you

<br/>
- Eirik - [github.com/clux](https://github.com/clux) - [clux.dev](https://clux.dev)
- Babylon Health - [babylonhealth.com](https://babylonhealth.com)

<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 0px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 140px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 280px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 420px" />
<img src="./babylon.png" style="border: 0px; width: 120px; position: absolute; top: 300px; left: 560px" />
