# Transcription planning and research

Start with the [technical implementation roadmap](transcription-implementation-plan.md). It targets Apple Silicon and macOS 26+, selects working defaults, and defines component contracts, audio ownership, resource limits, provider integration, implementation milestones, performance measurements, and release checks.

Its [UI design specification](transcription-implementation-plan.md#ui-design-specification) defines the black-and-white interface, layout, controls, and supplied [typeThis LED logo kit](assets/typeThis-led-kit.zip) ([preview](assets/typeThis-led-preview.png)). The logo retains its own gradient; interface controls remain monochrome. This kit replaces the earlier bot reference.

Supporting material:

- [Consolidated research and exploration](transcription-research-brief.md): product requirements, findings inventory, alternatives, and the questions that informed the roadmap.
- [TypeWhisper transcription pipeline audit](typewhisper-transcription-audit.md): 20 source-backed findings, compatibility and lifecycle lessons, and verification scenarios.
- [Earlier transcription design exploration](transcription-design.md): initial candidate choices and detailed reference behavior; use the roadmap for the current proposed implementation defaults.

Only documentation exists so far. Comparative performance and physical-device compatibility have not yet been measured. Grok CLI/OAuth speech access remains an explicit external feasibility question.
