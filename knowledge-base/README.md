# Knowledge base

The fictional business's information: services, starting prices, hours, appointment types, policies, the 10 most common questions and rules for the assistant.

- `ledgerlight.md`: Ledgerlight Tax & Advisory, a made-up CPA firm in Washington, DC. Names, prices, the address and the 555 phone number are all fictional.

Load it with `workflows/knowledge-base-ingestion.json`. To reload after edits, first delete the old chunks:

```sql
delete from knowledgebase where metadata->>'source' = 'ledgerlight-kb';
```
