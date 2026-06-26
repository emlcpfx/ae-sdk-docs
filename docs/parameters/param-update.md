# Param Update

> 1 Q&A · source: AE plugin dev community Discord

### Why doesn't PF_Cmd_UPDATE_PARAMS_UI properly update parameter values?

Two issues: (1) During UPDATE_PARAMS_UI, you should only change appearance (hidden, disabled, etc.), not values. Value changes don't play well with AE's instance application scheme. Set proper default values during PARAM_SETUP. (2) Never modify values directly on the original params array. Instead, make a copy of the param struct, modify the copy, and pass it back via PF_UpdateParamUI. Also ensure out_data->out_flags includes PF_OutFlag_REFRESH_UI. See the 'Supervisor' SDK sample project for correct implementation.

*Tags: `param-update`, `pf-update-param-ui`, `refresh-ui`, `supervisor-sample`, `update-params-ui`*

---
