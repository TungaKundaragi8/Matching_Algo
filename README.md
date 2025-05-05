1. reconciliation.component.ts

import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HttpClientModule } from '@angular/common/http';
import { ReconciliationService } from './reconciliation.service';
import { ReconciliationResult, Record } from './reconciliation.model';

@Component({
  selector: 'app-reconciliation',
  standalone: true,
  imports: [CommonModule, HttpClientModule],
  providers: [ReconciliationService],
  templateUrl: './reconciliation.component.html',
  styleUrls: ['./reconciliation.component.css']
})
export class ReconciliationComponent implements OnInit {
  matched: Record[] = [];
  unmatched: Record[] = [];

  constructor(private service: ReconciliationService) {}

  ngOnInit(): void {
    this.service.getReconciliationData().subscribe((data: ReconciliationResult) => {
      console.log('API Response:', data);
      this.matched = data.matched;
      this.unmatched = data.unmatched;
    });
  }
}


---

2. reconciliation.component.html

<h2>Matched Records ({{ matched.length }})</h2>
<table border="1" *ngIf="matched.length > 0">
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

<h2>Unmatched Records ({{ unmatched.length }})</h2>
<table border="1" *ngIf="unmatched.length > 0">
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

3. reconciliation.service.ts

import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { ReconciliationResult } from './reconciliation.model';

@Injectable()
export class ReconciliationService {
  private apiUrl = 'http://localhost:8080/api/reconcile'; // Update port if needed

  constructor(private http: HttpClient) {}

  getReconciliationData(): Observable<ReconciliationResult> {
    return this.http.get<ReconciliationResult>(this.apiUrl);
  }
}


---

4. reconciliation.model.ts

export interface Record {
  algoKey: string;
  starKey: string;
  status: string;
}

export interface ReconciliationResult {
  matched: Record[];
  unmatched: Record[];
}


---

5. Update main.ts to bootstrap this component

import { bootstrapApplication } from '@angular/platform-browser';
import { ReconciliationComponent } from './app/reconciliation/reconciliation.component';
import { provideHttpClient } from '@angular/common/http';

bootstrapApplication(ReconciliationComponent, {
  providers: [provideHttpClient()]
});
