Sure! Here's your full updated Angular code for the behavior:


---

✅ GOAL:

When you click hamburger menu, the white sidebar opens/closes.

When you click "System Setup" inside the white sidebar, it should:

Hide the white sidebar

Show the black sidebar (i.e., systemSetupSideBarVisible).




---

✅ 1. layout.component.html

<div class="layout">
  <app-header (sidebarToggle)="onSidebarToggle()"></app-header>

  <div class="main-area">
    <!-- White Sidebar -->
    <app-sidebar
      *ngIf="isWhiteSideBarVisible"
      [isCollapsed]="isCollapsed"
      (systemSetupClicked)="onSystemSetupClick()"
    ></app-sidebar>

    <!-- Black Sidebar -->
    <div *ngIf="isSystemSetUpSideBarVisible" class="sidebar-container">
      <!-- your black sidebar content here -->
    </div>

    <div class="content-area">
      <router-outlet></router-outlet>
    </div>
  </div>
</div>


---

✅ 2. layout.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'app-layout',
  templateUrl: './layout.component.html',
  styleUrls: ['./layout.component.css']
})
export class LayoutComponent {
  isCollapsed = true;
  isWhiteSideBarVisible = true;
  isSystemSetUpSideBarVisible = false;

  onSidebarToggle() {
    this.isCollapsed = !this.isCollapsed;
    this.isWhiteSideBarVisible = !this.isWhiteSideBarVisible;
    this.isSystemSetUpSideBarVisible = false;
  }

  onSystemSetupClick() {
    this.isWhiteSideBarVisible = false;
    this.isSystemSetUpSideBarVisible = true;
  }
}


---

✅ 3. header.component.html

<span (click)="toggleSidebar()">
  <svg xmlns="http://www.w3.org/2000/svg" height="25px" viewBox="0 0 960 960" width="25px" fill="#FFFFFF">
    <path d="..."/>
  </svg>
</span>


---

✅ 4. header.component.ts

import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-header',
  templateUrl: './header.component.html',
  styleUrls: ['./header.component.css']
})
export class HeaderComponent {
  @Output() sidebarToggle = new EventEmitter<void>();

  toggleSidebar(): void {
    this.sidebarToggle.emit();
  }
}


---

✅ 5. sidebar.component.html (White Sidebar)

<div class="main-sidebar-container" [ngClass]="{ 'collapsed': isCollapsed }">
  <ul class="menu">
    <li (click)="navigateToPath('inventory-configuration')">
      Inventory Configuration
    </li>

    <li (click)="navigateToPath('system-setup')">
      System Setup
    </li>

    <li (click)="navigateToPath('import-export')">
      Import Export Manager
    </li>

    <li (click)="navigateToPath('data-direction')">
      Data Directions
    </li>
  </ul>
</div>


---

✅ 6. sidebar.component.ts

import { Component, Input, Output, EventEmitter, OnInit } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';

@Component({
  selector: 'app-sidebar',
  templateUrl: './sidebar.component.html',
  styleUrls: ['./sidebar.component.css']
})
export class SidebarComponent implements OnInit {
  @Input() isCollapsed = false;
  @Output() systemSetupClicked = new EventEmitter<void>();

  constructor(private router: Router) {}

  ngOnInit(): void {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationEnd) {
        // Add route-based logic if needed
      }
    });
  }

  navigateToPath(path: string): void {
    if (path === 'system-setup') {
      this.systemSetupClicked.emit();
    } else {
      this.router.navigateByUrl(path);
    }
  }
}


---

✅ CSS (Optional)

You can add styling to .main-sidebar-container and .sidebar-container in CSS files to match your desired appearance.


---

Would you like me to zip and send a working Angular demo for this too?

