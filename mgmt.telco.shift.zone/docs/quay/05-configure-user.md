
## Get the local openshift-operators URL

```bash
oc get route -n openshift-operators | egrep "\-quay " | awk '{print $2}'
```
This URL **must** match the FQDN of the certificate you previously created.

On 1st login to the site you will need to create a user, select "Create Account" and complete the required fields for your user. Example below.

---
<p align="center">
  <img src="quay-config-4.png" alt="drawing" width="250"/>
</p>

---



---
<p align="center">
  <img src="quay-config-5.png" alt="drawing" width="250"/>
</p>

---

Once the account is created, login and create 2 "public" repositories names "ocp4811" and "ocpmetal4811" as illustrated below.


---
<p align="center">
  <img src="quay-config-6.png" alt="drawing" width="400"/>
</p>

---

---
<p align="center">
  <img src="quay-config-7.png" alt="drawing" width="400"/>
</p>

---

---
<p align="center">
  <img src="quay-config-8.png" alt="drawing" width="400"/>
</p>

---
