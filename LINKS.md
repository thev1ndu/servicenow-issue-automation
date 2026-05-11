# ServiceNow Instance Links

Replace the placeholders below with the actual URLs for your ServiceNow instance after setup. Keep this file private or gitignored if your instance URL is sensitive.

---

## Script Include

```
https://<your-instance>.service-now.com/now/nav/ui/classic/params/target/sys_script_include.do
```

Filter by name: `GitHubRepositoryDispatch`

---

## System Properties

```
https://<your-instance>.service-now.com/now/nav/ui/classic/params/target/sys_properties.do
```

Filter by name starting with: `github.dispatch`

---

## Scripted REST API

```
https://<your-instance>.service-now.com/now/nav/ui/classic/params/target/sys_ws_provider_list.do
```

Name: `GitHub Case Integration`

---

## Case table (for testing)

```
https://<your-instance>.service-now.com/now/nav/ui/classic/params/target/sn_customerservice_case_list.do
```

---

## Change Request table (for testing)

```
https://<your-instance>.service-now.com/now/nav/ui/classic/params/target/change_request_list.do
```
