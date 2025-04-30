1. main.ts

import { bootstrapApplication } from '@angular/platform-browser';
import { ReconciliationComponent } from './app/reconciliation/reconciliation.component';

bootstrapApplication(ReconciliationComponent);


---

2. reconciliation.component.ts

import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HttpClientModule } from '@angular/common/http';
import { ReconciliationService, ReconciliationResult } from './reconciliation.service';

@Component({
  selector: 'app-reconciliation',
  standalone: true,
  imports: [CommonModule, HttpClientModule],
  templateUrl: './reconciliation.component.html',
  styleUrls: ['./reconciliation.component.css']
})
export class ReconciliationComponent {
  matched: any[] = [];
  unmatched: any[] = [];

  constructor(private service: ReconciliationService) {}

  ngOnInit(): void {
    this.service.getReconciliationData().subscribe((data: ReconciliationResult) => {
      this.matched = data.matched;
      this.unmatched = data.unmatched;
    });
  }
}


---

3. reconciliation.component.html

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


---

4. reconciliation.component.css (optional styling)

table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 20px;
}
th, td {
  padding: 8px;
  text-align: left;
}
h2 {
  margin-top: 20px;
}


---

5. reconciliation.service.ts

import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Record {
  algoKey: string;
  starKey: string;
  status: string;
}

export interface ReconciliationResult {
  matched: Record[];
  unmatched: Record[];
}

@Injectable({
  providedIn: 'root'
})
export class ReconciliationService {
  private apiUrl = 'http://localhost:8080/api/reconcile';

  constructor(private http: HttpClient) {}

  getReconciliationData(): Observable<ReconciliationResult> {
    return this.http.get<ReconciliationResult>(this.apiUrl);
  }
}
