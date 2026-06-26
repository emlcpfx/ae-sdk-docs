# Parameter Update

> 1 Q&A · source: AE plugin dev community Discord

### How can I trigger a plugin refresh in Premiere when CEP updates hidden parameters, since Premiere doesn't treat script interactions as user actions like AE does?

There is no known workaround for this. Premiere does not consider script interactions as user actions, so PF_UserChangedParam won't be triggered from CEP. A practical workaround is to use external dependencies watching for file changes (though it's not very reactive) or add a button parameter to force a manual refresh.

*Tags: `cep`, `parameter-update`, `premiere`, `script-interaction`, `workaround`*

---
