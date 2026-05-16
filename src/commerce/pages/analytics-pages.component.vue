<template>
  <MainPageNavigation
      :navigationItems="navigationItems"
  />
  <div class="analytics-page">
    <div class="content-container">
      <div class="page-header">
        <div class="header-left">
          <h1>{{this.hotelName}}</h1>
          <p class="subtitle">{{ i18n.global.t('analytics.overview')}}</p>

          <div class="tabs">
            <div
                v-for="tab in tabsWithLabels"
                :key="tab.id"
                :class="['tab', { active: activeTab === tab.id }]"
                @click="changeTab(tab.id)"
            >
              {{ tab.label }}
            </div>
          </div>
        </div>

        <div class="header-stats">
          <!-- Card: Reservas esta semana -->
          <div class="stat-card stat-card--blue">
            <div class="stat-card__header">
              <span class="stat-card__title">RESERVAS ESTA SEMANA</span>
              <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
                <rect x="3" y="4" width="18" height="18" rx="2"/>
                <line x1="16" y1="2" x2="16" y2="6"/>
                <line x1="8" y1="2" x2="8" y2="6"/>
                <line x1="3" y1="10" x2="21" y2="10"/>
              </svg>
            </div>
            <div class="stat-card__body">
              <span class="stat-card__value">{{ weeklyBookingsCount }}</span>
              <svg class="stat-card__sparkline" viewBox="0 0 80 30" preserveAspectRatio="none">
                <polyline
                    points="0,25 10,20 20,22 30,10 40,14 50,8 60,12 70,5 80,8"
                    fill="none"
                    stroke="rgba(255,255,255,0.7)"
                    stroke-width="2"
                    stroke-linejoin="round"
                />
              </svg>
            </div>
            <div class="stat-card__footer">
              <span :class="weeklyBookingChange >= 0 ? 'trend-up' : 'trend-down'">
                {{ weeklyBookingChange >= 0 ? '+' : '' }}{{ weeklyBookingChange }}% vs. Last Week
              </span>
            </div>
          </div>

          <!-- Card: Clientes nuevos -->
          <div class="stat-card stat-card--white">
            <div class="stat-card__header">
              <span class="stat-card__title stat-card__title--dark">CLIENTES NUEVOS</span>
              <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="#2563eb" stroke-width="1.8">
                <circle cx="12" cy="8" r="4"/>
                <path d="M4 20c0-4 3.6-7 8-7s8 3 8 7"/>
                <circle cx="18" cy="7" r="3" fill="#2563eb" stroke="none"/>
                <path d="M16.5 7l1 1 2-2" stroke="white" stroke-width="1.5" fill="none"/>
              </svg>
            </div>
            <div class="stat-card__body">
              <span class="stat-card__value stat-card__value--dark">{{ monthlyNewClientsCount }}</span>
              <svg class="stat-card__donut" viewBox="0 0 36 36">
                <circle cx="18" cy="18" r="14" fill="none" stroke="#e5e7eb" stroke-width="4"/>
                <circle
                    cx="18" cy="18" r="14"
                    fill="none"
                    stroke="#2563eb"
                    stroke-width="4"
                    stroke-dasharray="87.96 87.96"
                    stroke-dashoffset="22"
                    transform="rotate(-90 18 18)"
                />
              </svg>
            </div>
            <div class="stat-card__footer">
              <span class="stat-card__footer--dark">Total This Month: {{ monthlyNewClientsCount }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="chart-container">
        <div v-if="activeDashboard.length">
          <LineChart
              :incomes="activeDashboard.map(e => e.totalIncome)"
              :expenses="activeDashboard.map(e => e.totalExpense)"
              :labels="formattedLabels"
              :t="i18n.global.t"
          />
        </div>
        <div v-else-if="isLoading" class="loading">
          {{ i18n.global.t('analytics.loading')}}
        </div>
        <div v-else class="no-data-placeholder">
          <div class="no-data-icon">
            <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
              <path d="M3 3v18h18"/>
              <path d="M18.7 8a4 4 0 0 0-7.4 0"/>
              <path d="M7 12v4h4"/>
              <path d="M7 8h.01"/>
            </svg>
          </div>
          <h3 class="no-data-title">{{ 'There aren\'t data for show it' }}</h3>
          <p class="no-data-message">
            {{ 'No existen datos por el momento para poder mostrar las analíticas. Estos se empezarán a mostrar cuando hayan reservas para el hotel.' }}
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { DashboardApiService } from "../services/dashboard-api.service.js";
import { Dashboard } from "../model/dashboard.entity.js";
import { HotelsApiService } from "../../shared/services/hotels-api.service.js";
import { BookingApiService } from "../../reservations/services/booking-api.service.js";
import LineChart from "../components/line-chart.vue";
import {Hotel} from "../../shared/model/hotel.entity.js";
import MainPageNavigation from "../../organizational-management/components/main-page-navigation.component.vue";
import OverviewIcon from "../../assets/organizational-management/overview-icon.svg";
import AnalyticsIcon from "../../assets/organizational-management/analytics-icon.svg";
import ProvidersIcon from "../../assets/organizational-management/providers-icon.svg";
import InventoryIcon from "../../assets/organizational-management/inventory-icon.svg";
import RoomsIcon from "../../assets/organizational-management/rooms-icon.svg";
import OrganizationIcon from "../../assets/organizational-management/organization-icon.svg";
import DevicesIcon from "../../assets/organizational-management/devices-icon.svg";
import ReservationsIcon from "../../assets/organizational-management/reservations-icon.svg";

import i18n from "../../i18n.js";
import { useAuthenticationStore } from '/src/iam/services/authentication.store.js';
const userId = useAuthenticationStore.state.userId;
export default {
  name: "AnalyticsPage",
  components: {MainPageNavigation, LineChart },
  data() {
    return {
      roleId: null,
      hotelName: '',
      hotelId: null,
      userId: userId,
      dashboardApi: new DashboardApiService(),
      hotelApi: new HotelsApiService(),
      bookingApi: new BookingApiService(),
      dashboard: [],
      monthlyDashboard: [],
      bookings: [],
      isLoading: false,
      navigationItems: [
        {id: "overview", label: "Overview", path: "", icon: OverviewIcon, isActive: true},
        {id: "analytics", label: "Analytics", path: "", icon: AnalyticsIcon, isActive: false},
        {id: "providers", label: "Providers", path: "", icon: ProvidersIcon, isActive: false},
        {id: "inventory", label: "Inventory", path: "", icon: InventoryIcon, isActive: false},
        {id: "rooms", label: "Rooms", path: "", icon: RoomsIcon, isActive: false},
        {id: "organization", label: "Organization", path: "", icon: OrganizationIcon, isActive: false},
        {id: "devices", label: "Devices", path: "", icon: DevicesIcon, isActive: false}
      ],
      activeTab: 'weekly',
      tabs: [
        { id: 'weekly' },
        { id: 'monthly' }
      ]
    };
  },

  mounted() {
    this.hotelId = this.$route.params.id || null;
    this.roleId = localStorage.getItem("roleId") || null;
    console.log("Hotel ID from route:", this.hotelId);
    this.loadNavigationItems();
  },

  computed: {
    i18n() {
      return i18n
    },
    tabsWithLabels() {
      return this.tabs.map(tab => {
        if (tab.id === 'weekly') {
          return { ...tab, label: this.i18n.global.t('analytics.line-chart.weekly') };
        }
        if (tab.id === 'monthly') {
          return { ...tab, label: this.i18n.global.t('analytics.line-chart.monthly') };
        }
        return tab;
      });
    },
    formattedLabels() {
      if (this.activeTab === 'weekly') {
        return this.dashboard.map((entry) => {
          const start = new Date(2025, 0, 1 + entry.weekNumbers * 7);
          const end = new Date(start);
          end.setDate(start.getDate() + 6);

          const format = (d) =>
              d.toLocaleDateString("en-GB", { day: "2-digit", month: "short" });

          return `${format(start)} - ${format(end)}`;
        });
      }

      if (this.activeTab === 'monthly') {
        return this.dashboard.map((entry) => {
          const start = new Date(2025, 0, 1 + entry.weekNumbers * 28);
          const end = new Date(start);
          end.setDate(start.getDate() + 27);

          const format = (d) =>
              d.toLocaleDateString("en-GB", { month: "short" });

          return `${format(end)}`;
        });
      }

      const months = Array(12).fill(0).map((_, i) =>
          new Date(2025, i).toLocaleDateString("en-GB", { month: "short" })
      );

      const counts = Array(12).fill(0); // totalIncome por mes

      this.monthlyDashboard.forEach((entry) => {
        const start = new Date(2025, 0, 1 + entry.weekNumbers * 7);
        const daysInWeek = Array.from({ length: 7 }, (_, i) => {
          const d = new Date(start);
          d.setDate(d.getDate() + i);
          return d.getMonth(); // 0 = Jan, 11 = Dec
        });

        const monthWithMostDays = daysInWeek
            .reduce((acc, m) => {
              acc[m] = (acc[m] || 0) + 1;
              return acc;
            }, {});

        const assignedMonth = Object.keys(monthWithMostDays)
            .map(Number)
            .reduce((a, b) => monthWithMostDays[a] >= monthWithMostDays[b] ? a : b);

        counts[assignedMonth] += entry.totalIncome;
      });

      return months;
    },
    activeDashboard() {
      return this.activeTab === 'monthly' ? this.monthlyDashboard : this.dashboard;
    },

    weeklyBookingsCount() {
      const { start, end } = this.currentWeekRange();
      return this.bookings.filter(b => {
        const d = new Date(b.startDate);
        return d >= start && d < end;
      }).length;
    },

    weeklyBookingChange() {
      const { start } = this.currentWeekRange();
      const lastStart = new Date(start);
      lastStart.setDate(lastStart.getDate() - 7);
      const lastEnd = new Date(start);
      const lastWeekCount = this.bookings.filter(b => {
        const d = new Date(b.startDate);
        return d >= lastStart && d < lastEnd;
      }).length;
      if (lastWeekCount === 0) return 0;
      return Math.round(((this.weeklyBookingsCount - lastWeekCount) / lastWeekCount) * 100);
    },

    monthlyNewClientsCount() {
      const now = new Date();
      const monthStart = new Date(now.getFullYear(), now.getMonth(), 1);
      const nextMonthStart = new Date(now.getFullYear(), now.getMonth() + 1, 1);
      const monthBookings = this.bookings.filter(b => {
        const d = new Date(b.startDate);
        return d >= monthStart && d < nextMonthStart;
      });
      return new Set(monthBookings.map(b => b.paymentCustomerId)).size;
    }
  },

  methods: {
    loadNavigationItems() {
      // update the path with the hotel ID

      if(this.roleId == 1) {
        // reactive navigation items for roleId 3
        console.log("Role ID is 3, setting navigation paths accordingly");
        this.navigationItems.forEach(item => {
          item.path = `/home/hotel/${this.hotelId}/${item.id}`;
        });
      }else if(this.roleId == 2) {
        console.log("Role ID is 2, setting navigation paths accordingly");
        const itemsAdmin = [
          {id: "overview", label: "Overview", path: `/home/hotel/${this.hotelId}/overview`, icon: OverviewIcon, isActive: true},
          {id: "analytics", label: "Analytics", path: `/home/hotel/${this.hotelId}/analytics`, icon: AnalyticsIcon, isActive: false},
          {id: "reservations", label: "Reservations", path: `/home/hotel/${this.hotelId}/reservations`, icon: ReservationsIcon, isActive: false},
          {id: "rooms", label: "Rooms", path: `/home/hotel/${this.hotelId}/rooms`, icon: RoomsIcon, isActive: false}
        ]

        this.navigationItems.splice(0, this.navigationItems.length, ...itemsAdmin);
      }
      try {
        this.navigationItems.forEach(item => {
          item.isActive = item.path === this.$route.path;
        });
      } catch (error) {
        console.error("Error loading navigation items:", error);
      }
    },
    async fetchWeeklyData() {
      try {
        this.isLoading = true;
        const res = await this.dashboardApi.getWeeklyData(this.hotelId);
        console.log("Respuesta del dashboard:", res.data);

        this.dashboard = res.data.map(entry =>
            Dashboard.fromDisplayableDashboard(entry)
        );
      } catch (error) {
        console.error("Error fetching weekly dashboard data:", error);
        this.dashboard = [];
      } finally {
        this.isLoading = false;
      }
    },

    async fetchMonthlyData() {
      try {
        this.isLoading = true;
        const res = await this.dashboardApi.getMonthlyData(this.hotelId);
        console.log("Respuesta del dashboard:", res.data);

        this.monthlyDashboard = res.data.map(entry =>
            Dashboard.fromDisplayableDashboard(entry)
        );
      } catch (error) {
        console.error("Error fetching monthly dashboard data:", error);
        this.monthlyDashboard = [];
      } finally {
        this.isLoading = false;
      }
    },

    changeTab(tabId) {
      this.activeTab = tabId;
      if (tabId === 'monthly' && this.monthlyDashboard.length === 0) {
        this.fetchMonthlyData();
      }
    },

    currentWeekRange() {
      const now = new Date();
      const dayOfWeek = now.getDay();
      const daysToMonday = dayOfWeek === 0 ? 6 : dayOfWeek - 1;
      const start = new Date(now);
      start.setDate(now.getDate() - daysToMonday);
      start.setHours(0, 0, 0, 0);
      const end = new Date(start);
      end.setDate(start.getDate() + 7);
      return { start, end };
    },

    async fetchBookings() {
      try {
        const res = await this.bookingApi.getBookings(this.hotelId);
        this.bookings = res.data || [];
      } catch (error) {
        console.error("Error fetching bookings:", error);
        this.bookings = [];
      }
    }
  },

  async created() {
    try {
      this.isLoading = true;
      const roleId = localStorage.getItem("roleId") || null;
      const hotelId = this.$route.params.id || null;
      const userId = localStorage.getItem("userId") || null;

      console.log(roleId, hotelId, userId);
      let hotel  = null;
      if(roleId == 1) {
        hotel = await HotelsApiService.getHotelByOwnerId(userId);
      }else if(roleId == 2) {
        hotel = await HotelsApiService.getHotelsById(hotelId);
      }

      if(!hotel) {
        console.log("No hotel found");
        return;
      }

      console.log("Hotel fetched:", hotel);

      this.hotelName = hotel.name;
      this.hotelId = hotelId;

      try {
        const res = await this.dashboardApi.getWeeklyData(this.hotelId);
        this.dashboard = res.data.map(entry =>
            Dashboard.fromDisplayableDashboard(entry)
        );
      } catch (error) {
        console.error("Error fetching dashboard data:", error);
        this.dashboard = [];
      }

      await this.fetchBookings();

    } catch (error) {
      console.error("Error fetching hotel:", error);
      this.hotelName = "Hotel Not Found";
    } finally {
      this.isLoading = false;
    }

  }
};
</script>

<style scoped>
.analytics-page {
  min-height: 100vh;
  padding: 0 2rem;
}

.content-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 0;
}

.page-header {
  margin-bottom: 2rem;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 2rem;
}

.header-left {
  flex: 1;
  min-width: 0;
}

.header-stats {
  display: flex;
  gap: 1rem;
  flex-shrink: 0;
  align-items: flex-start;
}

/* Stat Cards */
.stat-card {
  border-radius: 12px;
  padding: 1.1rem 1.3rem;
  width: 210px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.stat-card--blue {
  background: linear-gradient(135deg, #1d6fec 0%, #2563eb 60%, #1a56db 100%);
  color: #fff;
  box-shadow: 0 4px 16px rgba(37, 99, 235, 0.3);
}

.stat-card--white {
  background: #fff;
  border: 1.5px solid #e5e7eb;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.06);
}

.stat-card__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.stat-card__title {
  font-size: 0.65rem;
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.9);
}

.stat-card__title--dark {
  color: #374151;
}

.stat-card__body {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.stat-card__value {
  font-size: 2.4rem;
  font-weight: 700;
  line-height: 1;
  color: #fff;
}

.stat-card__value--dark {
  color: #111827;
}

.stat-card__sparkline {
  width: 80px;
  height: 30px;
  flex-shrink: 0;
}

.stat-card__donut {
  width: 48px;
  height: 48px;
  flex-shrink: 0;
}

.stat-card__footer {
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.85);
}

.stat-card__footer--dark {
  color: #6b7280;
}

.trend-up {
  color: rgba(255, 255, 255, 0.9);
  font-weight: 500;
}

.trend-down {
  color: #fca5a5;
  font-weight: 500;
}

h1 {
  font-size: 1.5rem;
  font-weight: 600;
  margin: 0;
  color: #333;
}

.subtitle {
  font-size: 0.9rem;
  color: #666;
  margin: 0.25rem 0 1.5rem 0;
}

.tabs {
  display: flex;
  border-bottom: 1px solid var(--gray-extra-light-color, #e0e0e0);
}

.tab {
  padding: 0.5rem 1rem;
  margin-right: 1.5rem;
  cursor: pointer;
  color: #666;
  position: relative;
}

.tab.active {
  color: #0066cc;
  font-weight: 500;
}

.tab.active::after {
  content: '';
  position: absolute;
  bottom: -1px;
  left: 0;
  right: 0;
  height: 2px;
  background-color: #0066cc;
}

.chart-container {
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  padding: 1.5rem;
  margin-top: 2rem;
  min-height: 600px;
  height: calc(100vh - 250px);
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.period-selector {
  display: flex;
  align-items: center;
  font-size: 0.9rem;
  color: #555;
  cursor: pointer;
}

.period-selector i {
  margin-left: 0.5rem;
  font-size: 0.8rem;
}

.legend {
  display: flex;
  align-items: center;
}

.legend-item {
  display: flex;
  align-items: center;
  margin-left: 1.5rem;
  font-size: 0.8rem;
  color: #666;
}

.color-indicator {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: 0.5rem;
}

.color-indicator.current {
  background-color: #0066cc;
}

.color-indicator.goal {
  background-color: #ff9999;
}

.chart {
  width: 100%;
  height: 300px;
  margin-bottom: 1rem;
}

.chart-placeholder {
  width: 100%;
  height: 100%;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: center;
  align-items: center;
}

.line-chart {
  width: 100%;
  height: 100%;
}

.chart-footer {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #888;
  padding-top: 0.5rem;
}

.date-range {
  text-align: center;
}

.loading, .error, .no-data {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 200px;
  width: 100%;
  color: #666;
  font-size: 0.9rem;
}

.error {
  color: #cc0000;
}

/* No Data Placeholder Styles */
.no-data-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 3rem 2rem;
  color: #666;
  height: 100%;
}

.no-data-icon {
  margin-bottom: 1.5rem;
  opacity: 0.6;
}

.no-data-icon svg {
  color: #999;
}

.no-data-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #333;
  margin: 0 0 1rem 0;
}

.no-data-message {
  font-size: 1rem;
  line-height: 1.6;
  color: #666;
  max-width: 500px;
  margin: 0;
}

.data-table {
  margin-top: 2rem;
  border-top: 1px solid #eee;
  padding-top: 1rem;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  padding: 0.75rem;
  text-align: left;
  border-bottom: 1px solid #eee;
}

th {
  font-weight: 600;
  color: #333;
  font-size: 0.9rem;
}

td {
  color: #555;
  font-size: 0.85rem;
}
</style>