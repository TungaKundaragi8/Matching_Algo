Thanks for the clarity! Here's exactly what you need:

✅ Resizable horizontal split-pane layout (like your image)
✅ Left pane: AG Grid for Accounting Period Groups
✅ Right pane:
  - AG Grid for Accounting Periods
  - Initially shows message: “Click on ➤ to view the accounting periods.”
✅ Both AG Grids have the same column layout


---

✅ Final Working Solution


---

🔧 1. Install angular-split and ag-grid

npm install angular-split ag-grid-angular ag-grid-community


---

📁 app.module.ts

import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AngularSplitModule } from 'angular-split';
import { AgGridModule } from 'ag-grid-angular';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    AngularSplitModule,
    AgGridModule.withComponents([]),
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}


---

🧩 app.component.html

<as-split direction="horizontal" [gutterSize]="5">
  <!-- LEFT PANE -->
  <as-split-area [size]="50">
    <div class="toolbar">
      <button (click)="onAdd()">Add</button>
      <button (click)="onDelete()">Delete</button>
      <button (click)="onRefresh()">Refresh</button>
    </div>

    <ag-grid-angular
      style="width: 100%; height: calc(100% - 40px);"
      class="ag-theme-alpine"
      [rowData]="groups"
      [columnDefs]="columnDefs"
      rowSelection="single"
      (rowClicked)="onGroupClick($event)">
    </ag-grid-angular>
  </as-split-area>

  <!-- RIGHT PANE -->
  <as-split-area [size]="50">
    <ng-container *ngIf="!selectedGroup">
      <div class="placeholder">
        Click on ➤ to view the accounting periods.
      </div>
    </ng-container>

    <ng-container *ngIf="selectedGroup">
      <ag-grid-angular
        style="width: 100%; height: 100%;"
        class="ag-theme-alpine"
        [rowData]="periods"
        [columnDefs]="columnDefs"
        rowSelection="multiple">
      </ag-grid-angular>
    </ng-container>
  </as-split-area>
</as-split>


---

🧠 app.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  selectedGroup: any = null;

  columnDefs = [
    { headerName: '', field: 'expand', width: 40, cellRenderer: () => '➤' },
    { headerName: 'Name', field: 'name', editable: true }
  ];

  groups = [
    { name: 'Q1 2025' },
    { name: 'Q2 2025' },
    { name: 'Q3 2025' }
  ];

  periods: any[] = [];

  onGroupClick(event: any) {
    this.selectedGroup = event.data;

    // Load dummy accounting periods
    this.periods = [
      { name: 'Jan 2025' },
      { name: 'Feb 2025' },
      { name: 'Mar 2025' }
    ];
  }

  onAdd() {
    alert('Add clicked');
  }

  onDelete() {
    alert('Delete clicked');
  }

  onRefresh() {
    alert('Refresh clicked');
  }
}


---

🎨 app.component.css

:host {
  display: block;
  height: 100vh;
}

.toolbar {
  padding: 8px;
  background-color: #f4f4f4;
}

.toolbar button {
  margin-right: 8px;
  padding: 6px 12px;
}

.placeholder {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-style: italic;
  font-size: 16px;
  color: #999;
}


---

✅ Outcome:

✔ Resizable split-pane (drag in middle)
✔ Left: Groups table
✔ Right: Same table structure (columns/rows) for periods
✔ Initial message until group is selected


---

Would you like the right grid to dynamically load from an API based on the selected group? I can also generate a full Angular + Spring Boot version if needed.

