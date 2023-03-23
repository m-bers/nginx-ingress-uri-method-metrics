# Per URI and Per Method Metrics for the NGINX Plus Ingress Controller

## Note:
This assumes you have already deployed the NGINX Plus Ingress Controller following the steps documented here (either via helm or via manifests): https://docs.nginx.com/nginx-ingress-controller/installation/

## Instructions

If you want to use NGINX+ Ingress Controller to dynamically insert status_zone directives for API monitoring at every location block within a particular type of resource in a way that differentiates between HTTP methods, Here's what to do: 

1. Go to the root of the project directory where you cloned the kubernetes-ingress repository. 
2. Copy any of your custom config (e.g. your helm `values.yml`) and any other user configuration to another directory outside of the project root. 
3. Checkout the branch of the repository that corresponds to the NGINX Ingress Controller release you are using. E.g. I'm on `v3.0.2`, so I ran 
```shell
cd kubernetes-ingress
git checkout v3.0.2
```
4. From there, go to `$PROJECT_ROOT/internal/configs`. The templates for global config and for Ingress resources are in the `version1` subfolder and the templates for CRDs are in the `version2` subfolder.
5. Determine which resource you want to apply per-URI and per-method metrics to and open up the file corresponding file with extension `.tmpl` For example, `$PROJECT_ROOT/internal/configs/version1/nginx-plus.ingress.tmpl` is the template file for the Ingress resource on NGINX Plus, and `$PROJECT_ROOT/internal/configs/version2/nginx-plus.transportserver.tmpl` is the template file for the TransportServer CRD resource on NGINX Plus.
6. Copy the contents of that file into one of two places, and make sure it's at the next level of indentation to the parent element, depending on if you deployed via helm or via manifests. 
  a. If via helm:
     The .tmpl file must be inserted at `controller.config.entries.[TEMPLATE_NAME]`
     For example, If you are modifying the Ingress template it would be at `controller.config.entries.ingress-template`
  b. If via manifests:
     The .tmpl file must be inserted at `data.[TEMPLATE_NAME]`
     For example with Ingress it would be `data.ingress-template`
6. If you have trouble figuring out where to insert the contents, see my [full example for helm](custom-values.yaml), and see [this one](https://github.com/nginxinc/kubernetes-ingress/tree/main/examples/shared-examples/custom-templates) for a basic example using the ConfigMap manifest.
7. Locate the portion of the template responsible for location block creation. On the Ingress template, you are looking for these specific lines:
```tmpl
{{range $location := $server.Locations}}
location {{$location.Path}} {
```
8. Insert the following below those lines:  
```tmpl 
if ($request_method = GET) {
  status_zone {{$location.ServiceName}}_GET;  
}
if ($request_method = POST) {
  status_zone {{$location.ServiceName}}_POST;  
}
```
9. Repeat for any other HTTP methods you want API metrics for.
10. Upgrade your helm release or apply the ConfigMap manifest, depending on your deployment method.
>Note: This will look slightly different for other resources besides Ingress. For example if you are using VirtualServer, you will need to instead locate these lines in the `.tmpl`:
```tmpl
{{ range $l := $s.Locations }}
location {{ $l.Path }} {
```
>And the lines you insert afterwards will need to use $l instead of $location, but everything else should be the same. 
