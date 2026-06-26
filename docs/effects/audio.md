# Audio

> 1 Q&A · source: AE plugin dev community Discord

### Why doesn't PF_Cmd_RENDER execute when an audio plugin uses PF_OutFlag_I_USE_AUDIO on a precomposed audio-only layer?

When a plugin sets PF_OutFlag_I_USE_AUDIO and the source layer is precomposed with no visual layers, After Effects may not trigger PF_Cmd_RENDER. The solution is to add PF_OutFlag_NON_PARAM_VARY to the global output flags in the plugin's global setup. This flag tells AE that the plugin's output varies based on non-parameter data (in this case, audio data), ensuring render commands are properly triggered even when the precomp contains only audio. Note: This flag is not required when the source layer is not precomposed, as AE automatically handles render triggering in that case.

*Tags: `aegp`, `audio`, `layer-checkout`, `params`, `render-loop`*

---
