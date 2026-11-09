<template>
	<div class="px-12 py-8 space-y-8">
		<div class="flex items-center">
			<FeatherIcon name="calendar" class="h-7 w-7 text-gray-500 mr-2.5" />
			<span class="font-semibold text-2xl text-gray-500 mr-2">Roster:</span>
			<span class="font-semibold text-2xl">{{ viewTitle }}</span>
			<div class="ml-auto space-x-2.5">
				<Dropdown
					:options="VIEW_OPTIONS"
					:button="{
						label: 'View',
						iconRight: 'chevron-down',
						size: 'md',
					}"
				>
				</Dropdown>
				<Dropdown
					:options="[
						{
							label: 'Shift Assignment',
							onClick: () => {
								showShiftAssignmentDialog = true;
							},
						},
					]"
					:button="{
						label: 'Create',
						variant: 'solid',
						iconRight: 'chevron-down',
						size: 'md',
					}"
				/>
			</div>
		</div>
		<MonthViewHeader
			:firstOfMonth="firstOfMonth"
			:dayFilter="dayFilter"
			:rangeFrom="rangeFrom"
			:rangeTo="rangeTo"
			:showRange="features.dateRange"
			:showDayFilter="features.dayFilter"
			@updateRange="(r: { from: string; to: string }) => { rangeFrom = r.from; rangeTo = r.to; }"
			@updateFilters="updateFilters"
			@addToMonth="addToMonth"
			@updateDay="(value: string) => (dayFilter = value)"
		/>
		<MonthViewTable
			v-if="isCompanySelected"
			ref="monthViewTable"
			:firstOfMonth="firstOfMonth"
			:employees="employees.data || []"
			:employeeFilters="employeeFilters"
			:shiftFilters="shiftFilters"
			:dayFilter="dayFilter"
			:rangeFrom="rangeFrom"
			:rangeTo="rangeTo"
			:allowWeekdayChange="features.weekdayChange"
		/>
		<div v-else class="py-40 text-center">Please select a company.</div>
	</div>
	<ShiftAssignmentDialog
		v-model="showShiftAssignmentDialog"
		:isDialogOpen="showShiftAssignmentDialog"
		:employees="employees.data"
		@fetchEvents="
			monthViewTable?.events.fetch();
			showShiftAssignmentDialog = false;
		"
	/>
</template>

<script setup lang="ts">
import { computed, ref, reactive } from "vue";
import { Dropdown, FeatherIcon, createListResource, createResource } from "frappe-ui";

import { dayjs, goTo, raiseToast } from "../utils";
import MonthViewTable from "../components/MonthViewTable.vue";
import MonthViewHeader from "../components/MonthViewHeader.vue";
import ShiftAssignmentDialog from "../components/ShiftAssignmentDialog.vue";

export type EmployeeFilters = {
	[K in "status" | "company" | "department" | "branch" | "designation"]?: string;
};
export type ShiftFilters = {
	[K in "shift_type" | "shift_location"]?: string;
};

const monthViewTable = ref<InstanceType<typeof MonthViewTable>>();
const isCompanySelected = ref(false);
const showShiftAssignmentDialog = ref(false);
// Which custom features HR Settings (Roster section) switches on
const rosterOptions = createResource({
	url: "et_adms.custom.weekday_shift.get_weekday_shift_options",
	auto: true,
	onSuccess(data: { date_range: number; day_filter: number }) {
		if (!data.date_range) rangeFrom.value = rangeTo.value = "";
		if (!data.day_filter) dayFilter.value = "";
	},
});
// Until the options arrive (or on a site without et_adms) behave like stock HRMS plus everything on
const features = computed(() => ({
	dateRange: rosterOptions.data ? !!rosterOptions.data.date_range : true,
	dayFilter: rosterOptions.data ? !!rosterOptions.data.day_filter : true,
	weekdayChange: rosterOptions.data ? !!rosterOptions.data.weekday_change : true,
}));
const dayFilter = ref("");
const rangeFrom = ref("");
const rangeTo = ref("");
const firstOfMonth = ref(dayjs().date(1).startOf("D"));
const employeeFilters = reactive<EmployeeFilters>({
	status: "Active",
});
const shiftFilters = reactive<ShiftFilters>({});

const VIEW_OPTIONS = [
	"Shift Type",
	"Shift Location",
	"Shift Assignment",
	"Shift Schedule",
	"Shift Schedule Assignment",
].map((label) => ({
	label,
	onClick: () => goTo(`/app/${label.toLowerCase().split(" ").join("-")}`),
}));

const WEEKDAYS = ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"];

// Says what the grid is showing, e.g. "Day View · Wed 25 Nov" or "Date Range · 01 Nov – 31 Dec 2027"
const viewTitle = computed(() => {
	if (rangeFrom.value && rangeTo.value)
		return `Date Range · ${dayjs(rangeFrom.value).format("DD MMM")} – ${dayjs(rangeTo.value).format("DD MMM YYYY")}`;
	if (dayFilter.value === "week") return "Weekly Pattern";
	if (dayFilter.value.startsWith("dow:"))
		return `Every ${WEEKDAYS[Number(dayFilter.value.split(":")[1])]}`;
	if (dayFilter.value.startsWith("date:"))
		return `Day View · ${dayjs(dayFilter.value.split(":")[1]).format("ddd DD MMM")}`;
	return "Month View";
});

const addToMonth = (change: number) => {
	firstOfMonth.value = firstOfMonth.value.add(change, "M");
	// Moving months goes back to the month view
	rangeFrom.value = rangeTo.value = "";
	// A picked date belongs to the month it was picked in; weekdays carry over
	if (dayFilter.value.startsWith("date:")) dayFilter.value = "";
};

const updateFilters = (newFilters: EmployeeFilters & ShiftFilters) => {
	isCompanySelected.value = !!newFilters.company;
	if (!isCompanySelected.value) return;
	let employeeUpdated = false;
	(Object.entries(newFilters) as [keyof EmployeeFilters | keyof ShiftFilters, string][]).forEach(
		([key, value]) => {
			if (["shift_type", "shift_location"].includes(key)) {
				if (value) shiftFilters[key] = value;
				else delete shiftFilters[key];
				return;
			}

			if (value) employeeFilters[key] = value;
			else delete employeeFilters[key];
			employeeUpdated = true;
		},
	);
	if (employeeUpdated) employees.fetch();
};

// RESOURCES

const employees = createListResource({
	doctype: "Employee",
	fields: ["name", "employee_name", "designation", "image"],
	filters: employeeFilters,
	pageLength: 99999,
	onError(error: { messages: string[] }) {
		raiseToast("error", error.messages[0]);
	},
});
</script>
