Here is the complete working code for both Spring Boot (backend) and Angular (frontend) to perform reconciliation of two CSV files and display matched/unmatched records in a clean tabular format.


---

SPRING BOOT BACKEND

1. Record.java

package com.example.reconciliation.model;

public class Record {
    private String algoKey;
    private String starKey;
    private String status;

    public Record(String algoKey, String starKey, String status) {
        this.algoKey = algoKey;
        this.starKey = starKey;
        this.status = status;
    }

    public String getAlgoKey() { return algoKey; }
    public void setAlgoKey(String algoKey) { this.algoKey = algoKey; }

    public String getStarKey() { return starKey; }
    public void setStarKey(String starKey) { this.starKey = starKey; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}


---

2. ReconciliationResult.java

package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private int matchedCount;
    private int unmatchedCount;

    public ReconciliationResult(List<Record> matched, List<Record> unmatched) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.matchedCount = matched.size();
        this.unmatchedCount = unmatched.size();
    }

    public List<Record> getMatched() { return matched; }
    public List<Record> getUnmatched() { return unmatched; }
    public int getMatchedCount() { return matchedCount; }
    public int getUnmatchedCount() { return unmatchedCount; }
}


---

3. ReconciliationService.java

package com.example.reconciliation.service;

import com.example.reconciliation.model.Record;
import com.example.reconciliation.model.ReconciliationResult;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.FileReader;
import java.util.*;

@Service
public class ReconciliationService {

    public ReconciliationResult reconcile(String algoPath, String starPath) {
        List<String> algoKeys = new ArrayList<>();
        List<String> starKeys = new ArrayList<>();
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        try {
            FileReader readerAlgo = new FileReader(algoPath);
            Iterable<CSVRecord> algoRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);
            for (CSVRecord record : algoRecords) {
                String key = record.get("Agreement_name")
                        .replace("_RIMR", "3CR")
                        .replace("_RIMP", "3CP")
                        .replaceAll("\\s+", "")
                        .trim();
                algoKeys.add(key);
            }

            FileReader readerStar = new FileReader(starPath);
            Iterable<CSVRecord> starRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);
            Map<String, List<Integer>> starKeyMap = new HashMap<>();

            int index = 0;
            for (CSVRecord record : starRecords) {
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim()
                        + "3" +
                        record.get("Post Direction").replaceAll("\\s+", "").trim();
                starKeys.add(key);
                starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(index);
                index++;
            }

            Set<Integer> matchedStarIndices = new HashSet<>();

            for (int i = 0; i < algoKeys.size(); i++) {
                String algoKey = algoKeys.get(i);
                List<Integer> matchIdx = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

                if (matchIdx.size() == 1 && !matchedStarIndices.contains(matchIdx.get(0))) {
                    matched.add(new Record(algoKey, starKeys.get(matchIdx.get(0)), "Match"));
                    matchedStarIndices.add(matchIdx.get(0));
                } else if (matchIdx.size() > 1) {
                    for (Integer idx : matchIdx) {
                        unmatched.add(new Record(algoKey, starKeys.get(idx), "Mismatch - Multiple STAR Keys"));
                    }
                } else {
                    unmatched.add(new Record(algoKey, "<No Match>", "Mismatch - No Match"));
                }
            }

            for (int i = 0; i < starKeys.size(); i++) {
                if (!matchedStarIndices.contains(i)) {
                    unmatched.add(new Record("<No Match>", starKeys.get(i), "Mismatch - No Match or Duplicate"));
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        return new ReconciliationResult(matched, unmatched);
    }
}


---

4. ReconciliationController.java

package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "http://localhost:4200")
public class ReconciliationController {

    @Autowired
    private ReconciliationService service;

    @GetMapping("/reconcile")
    public ReconciliationResult reconcile() {
        String algoPath = "C:/path/to/algo.csv";
        String starPath = "C:/path/to/star.csv";
        return service.reconcile(algoPath, starPath);
    }
}


---

ANGULAR FRONTEND

1. reconciliation.service.ts

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
  matchedCount: number;
  unmatchedCount: number;
}

@Injectable({
  providedIn: 'root'
})
export class ReconciliationService {
  private apiUrl = 'http://localhost:8080/api/reconcile';

  constructor(private http: HttpClient) {}

  getReconciliationResult(): Observable<ReconciliationResult> {
    return this.http.get<ReconciliationResult>(this.apiUrl);
  }
}


---

2. reconciliation.component.ts

import { Component, OnInit } from '@angular/core';
import { ReconciliationService, ReconciliationResult } from './reconciliation.service';

@Component({
  selector: 'app-reconciliation',
  templateUrl: './reconciliation.component.html'
})
export class ReconciliationComponent implements OnInit {
  result: ReconciliationResult | null = null;

  constructor(private reconciliationService: ReconciliationService) {}

  ngOnInit(): void {
    this.reconciliationService.getReconciliationResult().subscribe((data) => {
      this.result = data;
    });
  }
}


---

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
</div>


---

4. Add Bootstrap (Optional Styling)

In angular.json:

"styles": [
  "node_modules/bootstrap/dist/css/bootstrap.min.css",
  "src/styles.css"
]

Then run:

npm install bootstrap


---

Let me know if you'd like to:

Upload files from Angular and send to Spring Boot.

Deploy both together using Docker or a shared server.

Add filters/pagination/search to the UI.


