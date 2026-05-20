# Linq line spam guard — runbook summary

Protects shared iMessage lines from behavior that triggers carrier or provider flags.

## Principles

- Do not add a second Linq webhook ingress in this repo unless provider architecture changes—single ingress is intentional.  
- Rate-limit outbound bursts per line.  
- Monitor engagement metrics; low reply rate increases flag risk (see Q2 iMessage initiative).  

## When investigating flags

1. Check recent broadcast or automation volume per line  
2. Correlate with chatbot reply latency ([../../docs/chatbot/outbound-latency.md](../../docs/chatbot/outbound-latency.md))  
3. Review user engagement on flagged line vs healthy lines  

## Escalation

Coordinate with ops + imsg-service owners before rotating lines or changing provider config.

*Synthesized from proj-coach-backend Linq spam guard runbook source.*
