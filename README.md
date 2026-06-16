# claude-skill-public — Dev Agents

ชุด **Claude Code subagents** สำหรับงาน software development — แยกตาม stack/บทบาท เรียกใช้ด้วยชื่อใน Claude Code ได้เลย

## โครงสร้าง

```
agents/        # Subagent definitions — วางใน ~/.claude/agents/
```

## Agents

### Specialists (ตาม stack)
| Agent | บทบาท |
|-------|--------|
| `งู` | Python — Django / FastAPI / Flask / pandas / pytest |
| `ปลา` | Go — net/http, Gin, Fiber, Echo, gRPC, goroutines |
| `เสา` | Frontend — Next.js / React / TypeScript / Tailwind / shadcn |
| `ราก` | Backend Node — Express / NestJS / Fastify / Hono |
| `บ่อ` | Database — Supabase / Postgres / MongoDB / schema / RLS / index |
| `ใย` | API & integration — REST / GraphQL / webhook / OAuth / Stripe / LINE |
| `มือ` | Mobile — React Native / Expo / Flutter |
| `เมฆ` | DevOps — Docker / K8s / Terraform / AWS / GCP / CI/CD |
| `ตะแกรง` | Testing — Jest / Vitest / Pytest / Playwright / Cypress |
| `โล่` | Security audit — OWASP / CWE / CVE / injection / auth review |

### Senior coaches (review ก่อน/หลัง dev)
| Agent | บทบาท |
|-------|--------|
| `ภู` | Web coach — Frontend + Backend Node + API |
| `เขา` | Language coach — Python + Go |
| `ทะเล` | Data coach — schema / RLS / migration / query performance |
| `เกาะ` | Infra + Security coach — Docker / K8s / Terraform / CI |
| `แม่น้ำ` | Mobile + Test coach |

## วิธีใช้

```bash
git clone https://github.com/theryugi/claude-skill-public.git
cd claude-skill-public

# วาง agents (Windows: %USERPROFILE%\.claude\agents\)
cp agents/*.md ~/.claude/agents/
```

จากนั้นใน Claude Code spawn agent ด้วยชื่อได้เลย เช่น "ให้ `เสา` review component นี้" หรือ "`บ่อ` ออกแบบ schema ให้หน่อย"

> Specialist = ลงมือเขียน/แก้โค้ดตาม stack · Coach = review ออกแบบ/โค้ดก่อน-หลัง dev
