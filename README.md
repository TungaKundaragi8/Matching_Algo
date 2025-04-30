3. reconciliation.component.html

<div class="container mt-4" *ngIf="result">
  <h2>Reconciliation Report</h2>

  <div class="row mb-4">
    <div class="col-md-6">
      <div class="card bg-success text-white">
        <div class="card-body">
          <h5 class="card-title">Matched Records</h5>
          <p class="card-text">{{ result.matchedCount }}</p>
        </div>
      </div>
    </div>
    <div class="col-md-6">
      <div class="card bg-danger text-white">
        <div class="card-body">
          <h5 class="card-title">Unmatched Records</h5>
          <p class="card-text">{{ result.unmatchedCount }}</p>
        </div>
      </div>
    </div>
  </div>

  <h4>Matched Records</h4>
  <table class="table table-bordered">
    <thead>
      <tr>
        <th>ALGO Key</th>
        <th>STAR Key</th>
        <th>Status</th>
      </tr>
    </thead>
    <tbody>
      <tr *ngFor="let record of result.matched">
        <td>{{ record.algoKey }}</td>
        <td>{{ record.starKey }}</td>
        <td>{{ record.status }}</td>
      </tr>
    </tbody>
  </table>

  <h4>Unmatched Records</h4>
  <table class="table table-bordered">
    <thead>
      <tr>
        <th>ALGO Key</th>
        <th>STAR Key</th>
        <th>Status</th>
      </tr>
    </thead>
    <tbody>
      <tr *ngFor="let record of result.unmatched">
        <td>{{ record.algoKey }}</td>
        <td>{{ record.starKey }}</td>
        <td>{{ record.status }}</td>
      </tr>
    </tbody>
  </table>
</div
