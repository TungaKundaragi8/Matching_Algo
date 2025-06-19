To build the "Accounting Period Groups" screen using Angular with AG Grid as shown in your image, here’s a working example including:

Toolbar buttons (Add, Delete, Refresh)

AG Grid table

Sample data structure

Proper layout for the white content panel



---

✅ 1. Install AG Grid

If not installed:

npm install ag-grid-community ag-grid-angular

Import AG Grid in app.module.ts:

import { AgGridModule } from 'ag-grid-angular';

@NgModule({
  imports: [
    AgGridModule.withComponents([])
  ]
})
export class AppModule { }


---

✅ 2. accounting-period-groups.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'app-accounting-period-groups',
  templateUrl: './accounting-period-groups.component.html',
  styleUrls: ['./accounting-period-groups.component.css']
})
export class AccountingPeriodGroupsComponent {
  rowData = [
    { id: 'APG001', name: 'Q1 2025' },
    { id: 'APG002', name: 'Q2 2025' },
  ];

  columnDefs = [
    { headerName: 'ID', field: 'id', checkboxSelection: true },
    { headerName: 'Name', field: 'name' }
  ];

  onAdd() {
    alert('Add button clicked');
  }

  onDelete() {
    alert('Delete button clicked');
  }

  onRefresh() {
    alert('Refresh button clicked');
  }
}


---

✅ 3. accounting-period-groups.component.html

<div class="content-container">
  <div class="toolbar">
    <button (click)="onAdd()">Add</button>
    <button (click)="onDelete()">Delete</button>
    <button (click)="onRefresh()">Refresh</button>
  </div>

  <ag-grid-angular
    style="width: 100%; height: 400px;"
    class="ag-theme-alpine"
    [rowData]="rowData"
    [columnDefs]="columnDefs"
    rowSelection="multiple">
  </ag-grid-angular>

  <div class="accounting-periods-placeholder">
    Click on ➤ to view the accounting periods.
  </div>
</div>


---

✅ 4. accounting-period-groups.component.css

.content-container {
  padding: 16px;
  background-color: white;
  height: 100%;
}

.toolbar {
  margin-bottom: 12px;
}

.toolbar button {
  margin-right: 8px;
  padding: 6px 12px;
}

.accounting-periods-placeholder {
  margin-top: 20px;
  color: gray;
  font-style: italic;
}


---

✅ 5. Routing

Make sure your route is configured in app-routing.module.ts:

{ path: 'accounting-period-groups', component: AccountingPeriodGroupsComponent }


---

✅ 6. Usage in Sidebar Menu

In your sidebar menu:

<a routerLink="/accounting-period-groups">Accounting Period Groups</a>


---

If you want dynamic interaction with the right side (like loading "Accounting Periods" when selecting a group), I can also help you add a master-detail layout or two-column layout.

Would you like that functionality too?
