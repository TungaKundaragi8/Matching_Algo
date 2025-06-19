Great! You want a two-column layout where:

Left side has the table (AG Grid) showing Accounting Period Groups

Right side shows:
🔸 “Click on ➤ to view the accounting periods” (initially)
🔸 Or dynamically shows related accounting periods later



---

✅ Updated Layout Plan

▶ Left Pane:

Toolbar (Add, Delete, Refresh)

AG Grid table


▶ Right Pane:

Message: "Click on ➤ to view the accounting periods"



---

✅ Updated accounting-period-groups.component.html

<div class="main-split-layout">
  <!-- Left Panel -->
  <div class="left-panel">
    <div class="toolbar">
      <button (click)="onAdd()">Add</button>
      <button (click)="onDelete()">Delete</button>
      <button (click)="onRefresh()">Refresh</button>
    </div>

    <ag-grid-angular
      style="width: 100%; height: 100%;"
      class="ag-theme-alpine"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      rowSelection="single"
      (rowClicked)="onRowClick($event)">
    </ag-grid-angular>
  </div>

  <!-- Right Panel -->
  <div class="right-panel">
    <ng-container *ngIf="!selectedGroup">
      <div class="placeholder-text">
        Click on ➤ to view the accounting periods.
      </div>
    </ng-container>

    <ng-container *ngIf="selectedGroup">
      <h3>Accounting Periods for: {{ selectedGroup.name }}</h3>
      <!-- You can later load accounting period grid here -->
    </ng-container>
  </div>
</div>


---

✅ Add to accounting-period-groups.component.ts

selectedGroup: any = null;

onRowClick(event: any) {
  this.selectedGroup = event.data;
}


---

✅ Add to accounting-period-groups.component.css

.main-split-layout {
  display: flex;
  height: 100%;
  background-color: white;
}

.left-panel {
  width: 50%;
  padding: 16px;
  border-right: 1px solid #ccc;
  display: flex;
  flex-direction: column;
}

.right-panel {
  width: 50%;
  padding: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.toolbar {
  margin-bottom: 12px;
}

.toolbar button {
  margin-right: 8px;
  padding: 6px 12px;
}

.placeholder-text {
  color: gray;
  font-style: italic;
}


---

🔁 Optional Enhancement

Later, you can add another ag-grid-angular inside the right panel to load accounting periods for the selected group.


---

Let me know if you want the right panel to load a dynamic grid for accounting periods or a form for details.
