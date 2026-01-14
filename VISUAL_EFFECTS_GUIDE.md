# Visual Effects Guide (Enterprise AI Agents, Web + Mobile)

## 1) Top Visual Effects to Incorporate

1. **State Glow (Ambient Halo)**
   - Indicates agent state (idle, listening, processing, success).
2. **Subtle Pulse Ring**
   - Highlights microphone activity or voice capture.
3. **Waveform + Live Transcript**
   - Builds trust through visible input and agent understanding.
4. **Progressive Reveal**
   - Animates content cards in as the agent completes steps.
5. **Focus Depth (Parallax Lite)**
   - Adds depth without heavy motion or performance cost.
6. **Soft Shimmer (Processing)**
   - Signals the agent is working without blocking UI.
7. **Success Check + Micro-Confetti (Optional)**
   - For confirmed actions only; keep minimal in enterprise UX.
8. **Error Shake + Soft Red Outline**
   - Clear feedback for failures and validation errors.

## 2) Integration Instructions (Web Apps)

### A) Layering and Performance
- Use a dedicated overlay layer for effects.
- Keep animations under 300ms and avoid full-screen blurs.
- Prefer CSS transforms and opacity over layout changes.

### B) State Mapping
- Idle → static halo.
- Listening → pulse ring + waveform.
- Processing → shimmer + skeleton loading.
- Success → subtle checkmark pulse.
- Error → outline + brief shake.

### C) Accessibility
- Provide reduced-motion alternatives.
- Never rely on color alone for state.
- Provide textual state labels.

## 3) Integration Instructions (Mobile Apps)

### A) Platform Guidance
- iOS: Use Core Animation + haptic feedback for confirmations.
- Android: Use MotionLayout or Compose animations.

### B) Battery and Thermal Limits
- Prefer low-FPS transitions for idle states.
- Pause animations when app is backgrounded.

## 4) Integration Instructions (3D/Avatar Layers)

### A) State-Driven Animation
- Map state changes to subtle facial and posture shifts.
- Use 200–400ms easing for state transitions.

### B) Synchronization
- Drive animation from event bus signals.
- Keep a deterministic animation state machine.

## 5) Developer Notes (Recommended Defaults)

### Animation Durations
- Micro transitions: 120–180ms
- State changes: 200–400ms
- Loading shimmer: 800–1200ms loop

### Tokens
- `--fx-primary`: halo/pulse color
- `--fx-accent`: success color
- `--fx-error`: error color

### Event Hooks
- `agent.state.changed`
- `agent.voice.start`
- `agent.voice.end`
- `agent.action.success`
- `agent.action.error`
