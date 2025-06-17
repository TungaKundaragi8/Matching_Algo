sidebar.component.ts
ts
Copy code
import { Component, Input, OnChanges } from '@angular/core';

@Component({
  selector: 'app-sidebar',
  templateUrl: './sidebar.component.html',
  styleUrls: ['./sidebar.component.css']
})
export class SidebarComponent implements OnChanges {
  @Input() isCollapsed: boolean = false;

  selectedMainMenu: string | null = null;

  ngOnChanges() {
    if (this.isCollapsed) {
      // When toggling, reset submenu
      this.selectedMainMenu = null;
    }
  }

  openSubmenu(menu: string) {
    this.selectedMainMenu = menu;
    this.isCollapsed = true; // Close main sidebar when menu is clicked
  }

  backToMainMenu() {
    this.selectedMainMenu = null;
    this.isCollapsed = false; // Optionally re-open main sidebar on back
  }
}
🔸 sidebar.component.html
html
Copy code
<!-- MAIN SIDEBAR -->
<div class="mainsidebar-container" *ngIf="!isCollapsed && !selectedMainMenu">
  <ul>
    <li (click)="openSubmenu('inventory')">Inventory Configuration</li>
    <li (click)="openSubmenu('system')">System Setup</li>
    <li (click)="openSubmenu('admin')">Admin</li>
    <li (click)="openSubmenu('reports')">Reports</li>
  </ul>
</div>

<!-- SUBMENU SIDEBAR -->
<div class="sidebar-container" *ngIf="selectedMainMenu">
  <button class="back-btn" (click)="backToMainMenu()">⬅ Back</button>
  <ng-container [ngSwitch]="selectedMainMenu">
    
    <!-- Inventory Submenu -->
    <div *ngSwitchCase="'inventory'">
      <ul>
        <li>Manage Dashboard</li>
        <li>Task Automation</li>
        <li>Recon Manager</li>
      </ul>
    </div>

    <!-- System Setup Submenu -->
    <div *ngSwitchCase="'system'">
      <ul>
        <li>Manage Company</li>
        <li>Recon Master</li>
      </ul>
    </div>

    <!-- Add more submenus similarly -->
    
  </ng-container>
</div>
🔸 sidebar.component.css
css
Copy code
.mainsidebar-container {
  width: 250px;
  background: white;
  height: 100vh;
  padding: 1rem;
  border-right: 1px solid #ccc;
}

.sidebar-container {
  width: 250px;
  background: lightgreen;
  height: 100vh;
  padding: 1rem;
  position: absolute;
  left: 0;
  top: 0;
  z-index: 100;
}

ul {
  list-style: none;
  padding: 0;
}

li {
  padding: 0.75rem 0;
  cursor: pointer;
  font-weight: bold;
}

.back-btn {
  background: none;
  border: none;
  color: black;
  font-size: 1rem;
  margin-bottom: 1rem;
  cursor: pointer;
}
🔸 layout.component.ts (where toggle is)
ts
Copy code
export class LayoutComponent {
  isCollapsed = false;

  onSidebarToggle() {
    this.isCollapsed = !this.isCollapsed;
  }
}
🔸 layout.component.html
html
Copy code
<app-header (sidebarToggle)="onSidebarToggle()"></app-header>

<app-sidebar [isCollapsed]="isCollapsed"></app-sidebar>

<router-outlet></router-outlet>
✅ Summary of Behavior:
Event	What happens
🔘 Toggle Button	Toggles isCollapsed. Shows/hides main sidebar only.
📄 Click "Inventory Configuration"	Main sidebar closes, green submenu opens with "Manage Dashboard", etc.
📄 Click "System Setup"	Same – closes main, opens submenu.
⬅ Back Button	Goes back to main sidebar.

Let me know if you’d like:

Animations for transitions

PrimeNG styling applied

Routing on submenu clicks

I can enhance it further!








