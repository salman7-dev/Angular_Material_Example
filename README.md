7. Client Summary Index
Agreed structure: owners/{ownerId}/summary_client_index/{year}/{month}/clients/{clientId}
Recommended fields:
clientId
ownerId
yearMonth
receivable
advance
status
The index is derived data, not the detailed ledger source. It exists so a September drill-down can query matching client positions instead of scanning every client in application code. A for-loop may process returned matches, but the application should not loop through all 1,000+ clients.
