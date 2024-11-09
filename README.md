# CI/CD for cloud-native GenAI reference architecture

This repo will show you how to create a complete CI/CD architecture for GenAI apps.

This example will run entirely locally on a `kind` cluster on your laptop.
In development, we will use an external LLM `together.ai`.
Helix will provide the versioned AI app implementation (prompt management, knowledge/RAG and API integrations) and evals (testing). Flux will manage deployment.

![Reference Architecture](ref.png)

There are two main flows: the CI (testing) flow where you can run `helix test` locally or in CI.

# Setup

## 1. Fork this repo

Start by forking this repo. This is because part of the workflow is pushing changes to the repo and having GitHub Actions and Flux react to changes, so you'll need write access to the repo.

Then check out the repo on your local machine:

```
git clone git@github.com:<yourusername>/genai-cicd-ref
cd genai-cicd-ref
```

## 2. Install helix in kind

Requirements:
* [docker](https://www.docker.com/)
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) or `brew install kind`
* [kubectl](https://kubernetes.io/docs/tasks/tools/) or `brew install kubectl`
* [helm](https://helm.sh/docs/intro/install/) or `brew install helm`
* [flux cli](https://fluxcd.io/flux/installation/) or `brew install fluxcd/tap/flux`

We will run the `kind_helm_install.sh` script which will install helix in kind via helm.

For this deployment, to simplify things, we'll use [Together.ai](https://together.ai) as an external LLM provider (which provides free credit for new accounts), but you can later attach a Helix GPU runner [in Kubernetes](https://docs.helix.ml/helix/private-deployment/manual-install/kubernetes/#deploying-a-runner) or [otherwise](https://docs.helix.ml/helix/private-deployment/manual-install/).

```
export HELIX_VERSION=1.4.0-rc3
export TOGETHER_API_KEY=<your-together-key>
bash kind_helm_install.sh
```

```
watch kubectl get po
```

Should show helix starting up and running in your local kind cluster.
Once all the pods are running, `ctrl+c` the `watch` and run the four commands the script printed at the end of the install to start a port-forward session. Leave that running.

Load [http://localhost:8080](http://localhost:8080) and you should see Helix. It takes a few minutes to boot.

Register for a new account (in your local helix install, through the web interface) and log in.

Install the aispec CRDs and start the Helix Kubernetes Operator. For now we do this by cloning the helix repo, but these will be properly packaged and released as container images soon. In a new terminal session (you will need go installed - e.g `brew install go`):

```
git clone https://github.com/helixml/helix
cd helix/operator
make install
```

Go to your helix account page (click the ... button in the bottom left and go to Account & API section) then copy and paste the `export` commands for `HELIX_URL` and `HELIX_API_KEY` from the "Set authentication credentials" section. Run them, then run the Helix Kubernetes Operator:

```
make run
```

Leave the operator running in this terminal window. You should have two terminal windows now: one with the `port-forward` running in it and another with the helix operator running in it.

Test that the operator is working by deploying an aispec just with `kubectl` in a new terminal window:
```
kubectl apply -f aispecs/money.yaml
```

It should look like this:

![3 terminals showing portforward and operator running side by side](3-terminals.png)

Inside helix, the app should now be working. Go to the app store on the homepage, then launch the money app:

![Screenshot of Exchange Rates Chatbot](exchangerates.png)

You can use it to query live currency exchange rates.

Clean up the app:
```
kubectl delete -f aispecs/money.yaml
```

## 3. Install Flux

We will use Flux to automate GitOps deployments of changes to this app, rather than manually using `kubectl`.

Install flux in the kind cluster:
```
flux install
```

Add your fork of this repo to flux:

```
flux create source git aispecs \
    --url=https://github.com/<yourusername>/genai-cicd-ref \
    --branch=main
```

Set up flux to reconcile aispecs in your fork:
```
flux create kustomization aispecs \
    --source=GitRepository/aispecs \
    --path="./aispecs" --prune=true \
    --interval=1m --target-namespace=default
```

[TODO: set up ngrok and set up env vars in GHA so that CI runs against local cluster.]

# Continuous Integration: Testing

Go to your helix account page (click the ... button in the bottom left and go to Account & API section, then copy and paste the `export` commands for `HELIX_URL` and `HELIX_API_KEY`).

```
git checkout -b new-feature
```

Edit the aispec `aispec/money.yaml` to add a feature or test.

Run tests locally:
```
helix test -f aispecs/money.yaml
```

Push to CI:
```
git commit -am "update"
git push
```

(CI won't work unless you configure ngrok which is outside of the scope of this tutorial for now.)

# Continuous Delivery: Deployment via GitOps

If the tests are green, you can merge to main.
On push to main, Flux will pick up the new manifest and deploy it to your cluster.

You can run:
```
flux get kustomizations --watch
```
Flux can take up to a minute to notice the change in the repo.

Open the app in your browser by navigating to the "App Store" in your local helix install web UI, and observe the new improved GenAI capabilities!
