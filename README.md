<h2>Matched Records</h2>
<table border="1" *ngIf="matched.length">
  <thead>
    <tr>
      <th>Algo Key</th>
      <th>Star Key</th>
      <th>Status</th>
    </tr>
  </thead>
  <tbody>
    <tr *ngFor="let record of matched">
      <td>{{ record.algoKey }}</td>
      <td>{{ record.starKey }}</td>
      <td>{{ record.status }}</td>
    </tr>
  </tbody>
</table>

<h2>Unmatched Records</h2>
<table border="1" *ngIf="unmatched.length">
  <thead>
    <tr>
      <th>Algo Key</th>
      <th>Star Key</th>
      <th>Status</th>
    </tr>
  </thead>
  <tbody>
    <tr *ngFor="let record of unmatched">
      <td>{{ record.algoKey }}</td>
      <td>{{ record.starKey }}</td>
      <td>{{ record.status }}</td>
    </tr>
  </tbody>
</table>
