# CARD-16d — Validação end-to-end e correlação

**Tipo:** Validação  
**Status:** To Do  
**Depende de:** CARD-16c (stack K8s no ar)  
**Bloqueia:** nenhum  
**Parte de:** [CARD-16](CARD-16-observabilidade.md)

---

## Contexto

Com a stack completa no ar e o código instrumentado, este card valida que os três pilares estão funcionando e — mais importante — que é possível **correlacionar** um problema entre eles: partir de uma métrica anômala, chegar no trace, e do trace encontrar os logs.

---

## Critérios de aceite

### Traces (Jaeger)
- [ ] Trace de `POST /ordens-servico` visível no Jaeger com spans filhos de SQL
- [ ] Span do consumer do Estoque aparece como filho do span do Atendimento (mesmo TraceId propagado via RabbitMQ)
- [ ] Duração de cada span visível (HTTP, SQL, RabbitMQ)

### Métricas (Prometheus + Grafana)
- [ ] Métricas `http_server_duration_*` visíveis no Prometheus para Atendimento e Estoque
- [ ] Dashboard no Grafana com pelo menos: requisições/s, latência p95, taxa de erro
- [ ] Dashboard exportado como JSON e commitado em `k8s/observabilidade/grafana/`

### Logs (Loki + Grafana)
- [ ] Logs dos pods `atendimento` e `estoque` visíveis no Grafana (datasource Loki)
- [ ] Campo `TraceId` presente nos logs JSON
- [ ] Filtro por `{namespace="oficina-mecanica"}` retorna logs de ambos os serviços

### Correlação
- [ ] A partir de um log com `TraceId`, conseguir abrir o trace correspondente no Jaeger
- [ ] A partir de um trace no Jaeger, conseguir filtrar logs do mesmo `TraceId` no Loki

---

## Roteiro de validação

### 1. Verificar pods no ar
```bash
kubectl get pods -n observabilidade
# Esperado: jaeger, prometheus, loki, promtail, grafana todos Running
```

### 2. Fazer requisição de teste
```bash
# Usar a collection do Postman ou rodar o fluxo completo de OS
# (os spans serão gerados automaticamente pelo OTel)
```

### 3. Validar traces no Jaeger
```
Abrir: http://localhost:30086
Service: OficinaMecanica.Atendimento
Operation: POST /ordens-servico
→ Deve mostrar o trace com spans filhos de SQL e RabbitMQ
→ O span do Estoque deve aparecer conectado ao mesmo TraceId
```

### 4. Validar métricas no Prometheus
```
Abrir: http://localhost:30090
Query: http_server_request_duration_seconds_count{job="atendimento"}
→ Deve retornar valores crescentes a cada requisição
```

### 5. Validar logs no Grafana (Loki)
```
Abrir: http://localhost:30300
Explore → Datasource: Loki
Query: {namespace="oficina-mecanica", app="atendimento"}
→ Deve mostrar logs JSON com TraceId
```

### 6. Correlacionar trace ↔ log
```
No Loki: copiar o TraceId de um log
No Jaeger: colar o TraceId no campo de busca
→ Deve abrir o trace exato correspondente ao log
```

---

## Passos

1. Confirmar todos os pods `Running` nos dois namespaces
2. Executar fluxo completo de OS (Postman ou script)
3. Validar trace no Jaeger (critérios acima)
4. Validar métricas no Prometheus
5. Validar logs no Grafana/Loki
6. Fazer correlação manual trace ↔ log
7. Criar dashboard no Grafana e exportar JSON
8. Commitar dashboard JSON em `k8s/observabilidade/grafana/dashboard-oficina.json`
