<template>
	<div
		class="rounded-lg border overflow-auto max-h-[45rem]"
		:class="loading && 'animate-pulse pointer-events-none'"
	>
		<table class="border-separate border-spacing-0">
			<thead>
				<tr class="sticky top-0 bg-white z-10">
					<!-- Employee Search -->
					<th class="p-2 border-b">
						<Autocomplete
							:options="employeeSearchOptions"
							v-model="employeeSearch"
							placeholder="Search Employee"
							:multiple="true"
						/>
					</th>

					<!-- Day/Date Row -->
					<th
						v-for="(day, idx) in daysOfMonth"
						:key="idx"
						class="font-medium border-b"
						:class="{ 'border-l': idx, 'min-w-[22rem]': daysOfMonth.length === 1 }"
					>
						<button
							v-if="isWeekly && props.allowWeekdayChange"
							class="px-3 py-1 rounded hover:bg-gray-100"
							:title="`Change every ${day.fullDayName}`"
							@click="openWeekday(day)"
						>
							Every {{ day.fullDayName }}
						</button>
						<span v-else-if="isWeekly">Every {{ day.fullDayName }}</span>
						<span v-else :title="day.fullDayName + ' ' + dayjs(day.date).format('DD MMM YYYY')">
							{{ day.dayName }} {{ dayjs(day.date).format(hasRange ? "DD MMM" : "DD") }}
						</span>
					</th>
				</tr>
			</thead>
			<tbody>
				<tr v-for="(employee, rowIdx) in employees" :key="employee.name">
					<!-- Employee Column -->
					<td
						v-if="
							!employeeSearch?.length ||
							employeeSearch?.some((item) => item.value === employee?.name)
						"
						class="px-2 py-7 z-[5]"
						:class="{ 'border-t': rowIdx }"
					>
						<div class="flex" :class="!employee.designation && 'items-center'">
							<Avatar
								:label="employee.employee_name"
								:image="employee.image"
								size="2xl"
							/>
							<div class="flex flex-col ml-2 my-0.5 truncate">
								<div class="truncate text-base font-medium">
									{{ employee.employee_name }}
								</div>
								<div class="mt-auto text-xs text-gray-500 truncate">
									{{ employee.designation }}
								</div>
							</div>
						</div>
					</td>

					<!-- Events -->
					<td
						v-if="
							!employeeSearch?.length ||
							employeeSearch?.some((item) => item.value === employee?.name)
						"
						v-for="(day, colIdx) in daysOfMonth"
						:key="colIdx"
						class="p-1.5"
						:class="{
							'border-l': colIdx,
							'border-t': rowIdx,
							'align-top': cell(employee.name, day),
							'align-middle bg-blue-50':
								cell(employee.name, day)?.holiday,
							'align-middle bg-pink-50':
								cell(employee.name, day)?.leave,
							'bg-gray-50':
								dropCell.employee === employee.name &&
								dropCell.date === day.date &&
								!(
									isHolidayOrLeave(employee.name, day.date) ||
									hasSameShift(employee.name, day.date)
								),
						}"
						@mouseenter="
							hoveredCell.employee = employee.name;
							hoveredCell.date = day.date;
						"
						@mouseleave="
							hoveredCell.employee = '';
							hoveredCell.date = '';
						"
						@dragover.prevent
						@dragenter="
							dropCell.employee = employee.name;
							dropCell.date = day.date;
						"
						@drop="
							() => {
								if (
									!(
										isHolidayOrLeave(employee.name, day.date) ||
										hasSameShift(employee.name, day.date)
									)
								) {
									loading = true;
									swapShift.submit();
								}
							}
						"
					>
						<!-- Holiday -->
						<div v-if="cell(employee.name, day)?.holiday" class="blocked-cell">
							<div
								v-html="
									cell(employee.name, day).weekly_off
										? '<strong>WO</strong>'
										: cell(employee.name, day).description
								"
							></div>
						</div>

						<!-- Leave -->
						<div v-else-if="cell(employee.name, day)?.leave" class="blocked-cell">
							{{ cell(employee.name, day).leave_type }}
						</div>

						<!-- Shifts -->
						<div v-else class="flex flex-col space-y-1.5 translate-x-0 translate-y-0">
							<div
								v-for="shift in cell(employee.name, day)"
								@mouseenter="
									hoveredCell.shift = shift.name;
									hoveredCell.shift_type = shift.shift_type;
									hoveredCell.shift_location = shift.shift_location;
									hoveredCell.shift_status = shift.status;
								"
								@mouseleave="
									hoveredCell.shift = '';
									hoveredCell.shift_type = '';
									hoveredCell.shift_location = '';
									hoveredCell.shift_status = '';
								"
								@dragenter="dropCell.shift = shift.name"
								@dragleave="dropCell.shift = ''"
								:draggable="true"
								@dragstart="
									(event) => {
										if (event.dataTransfer) {
											event.dataTransfer.effectAllowed = 'move';
										}
									}
								"
								@dragend="
									if (!loading) dropCell = { employee: '', date: '', shift: '' };
								"
								class="rounded border-2 p-2 cursor-pointer"
								:class="[
									shift.status === 'Inactive' && 'border-dashed',
									dropCell.employee === employee.name &&
										dropCell.date === day.date &&
										dropCell.shift === shift.name &&
										!hasSameShift(employee.name, day.date) &&
										'scale-105',
									hoveredCell.employee === employee.name &&
										hoveredCell.date === day.date &&
										hoveredCell.shift === shift.name &&
										dropCell.employee &&
										'opacity-0',
								]"
								:style="{
									borderColor:
										hoveredCell.shift === shift.name &&
										hoveredCell.date === day.date
											? colors[shift.color as Color][300]
											: colors[shift.color as Color][200],
									backgroundColor:
										shift.status === 'Active'
											? colors[shift.color as Color][50]
											: 'white',
								}"
								@click="
									isWeekly && props.allowWeekdayChange
										? openWeekday(day, employee.name)
										: ((shiftAssignment = shift.name),
											(showShiftAssignmentDialog = true))
								"
							>
								<div
									class="truncate mb-1.5 pointer-events-none text-base font-medium"
								>
									{{ shift["shift_type"] }}
									<span v-if="shift.count > 1" class="text-xs text-gray-500 font-normal">
										× {{ shift.count }} days
									</span>
								</div>
								<div class="text-xs text-gray-500 pointer-events-none space-y-1.5">
									<div class="flex items-center space-x-1">
										<FeatherIcon
											name="clock"
											class="stroke-gray-400"
											style="
												 {
													height: 0.82rem;
													width: 0.82rem;
												}
											"
										/>
										<span>
											{{ shift["start_time"] }} - {{ shift["end_time"] }}
										</span>
									</div>
									<div
										v-if="shift['shift_location']"
										class="flex items-center space-x-1"
									>
										<FeatherIcon
											name="map-pin"
											class="stroke-gray-400"
											style="
												 {
													height: 0.82rem;
													width: 0.82rem;
												}
											"
										/>
										<span>{{ shift["shift_location"] }}</span>
									</div>
								</div>
							</div>

							<!-- Add Shift -->
							<Button
								variant="outline"
								icon="plus"
								class="border-2 active:bg-white w-full"
								:class="
									hoveredCell.employee === employee.name &&
									hoveredCell.date === day.date &&
									!dropCell.employee
										? 'visible'
										: 'invisible'
								"
								@click="
									isWeekly && props.allowWeekdayChange
										? openWeekday(day, employee.name)
										: ((shiftAssignment = ''), (showShiftAssignmentDialog = true))
								"
							/>
						</div>
					</td>
				</tr>
			</tbody>
		</table>
	</div>

	<WeekdayShiftDialog
		v-model="weekdayDialog.show"
		:weekday="weekdayDialog.weekday"
		:weekdayName="weekdayDialog.name"
		:employees="weekdayDialog.employee ? [weekdayDialog.employee] : visibleEmployees"
		:employeeLabels="dialogEmployeeLabels"
		:defaultFrom="hasRange ? props.rangeFrom : ''"
		:defaultTo="hasRange ? props.rangeTo : ''"
		@applied="events.fetch()"
	/>
	<ShiftAssignmentDialog
		v-model="showShiftAssignmentDialog"
		:isDialogOpen="showShiftAssignmentDialog"
		:shiftAssignmentName="shiftAssignment"
		:selectedCell="{ employee: hoveredCell.employee, date: hoveredCell.date }"
		:employees="employees"
		@fetchEvents="
			events.fetch();
			showShiftAssignmentDialog = false;
		"
	/>
</template>

<script setup lang="ts">
import { ref, computed, watch } from "vue";
import colors from "tailwindcss/colors";
import { Avatar, Autocomplete, FeatherIcon, createResource } from "frappe-ui";
import { Dayjs } from "dayjs";

import { dayjs, raiseToast } from "../utils";
import { EmployeeFilters, ShiftFilters } from "../views/MonthView.vue";
import ShiftAssignmentDialog from "./ShiftAssignmentDialog.vue";
import WeekdayShiftDialog from "./WeekdayShiftDialog.vue";

interface Holiday {
	holiday: string;
	description: string;
	weekly_off: 0 | 1;
}

interface HolidayWithDate extends Holiday {
	holiday_date: string;
}

interface Leave {
	leave: string;
	leave_type: string;
}

interface LeaveApplication extends Leave {
	from_date: string;
	to_date: string;
}

type Color =
	| "blue"
	| "cyan"
	| "fuchsia"
	| "green"
	| "lime"
	| "orange"
	| "pink"
	| "red"
	| "violet"
	| "yellow";

type Shift = {
	[K in "name" | "shift_type" | "status" | "start_time" | "end_time" | "shift_location"]: string;
} & {
	color: Color;
};

interface ShiftAssignment extends Shift {
	start_date: string;
	end_date: string;
}

type Events = Record<string, (HolidayWithDate | LeaveApplication | ShiftAssignment)[]>;
type MappedEvents = Record<string, Record<string, Holiday | Leave | Shift[]>>;

const props = defineProps<{
	firstOfMonth: Dayjs;
	employees: {
		[K in "name" | "employee_name" | "designation" | "image"]: string;
	}[];
	employeeFilters: { [K in keyof EmployeeFilters]?: string };
	shiftFilters: { [K in keyof ShiftFilters]?: string };
	// "" = every day, "dow:<0-6>" = that weekday only, "date:<YYYY-MM-DD>" = that date only
	dayFilter?: string;
	// Custom From/To range; when both are set it replaces the month
	rangeFrom?: string;
	rangeTo?: string;
	// HR Settings → Roster: may a weekday header / tile open the weekday dialog
	allowWeekdayChange?: boolean;
}>();

const loading = ref(true);
const employeeSearch = ref<{ value: string; label: string }[]>();
const shiftAssignment = ref<string>();
const showShiftAssignmentDialog = ref(false);
const hoveredCell = ref({
	employee: "",
	date: "",
	shift: "",
	shift_type: "",
	shift_location: "",
	shift_status: "",
});
const dropCell = ref({ employee: "", date: "", shift: "" });

const hasRange = computed(
	() => !!props.rangeFrom && !!props.rangeTo && !dayjs(props.rangeTo).isBefore(props.rangeFrom),
);
const rangeStart = computed(() => (hasRange.value ? dayjs(props.rangeFrom) : props.firstOfMonth));
const rangeEnd = computed(() =>
	hasRange.value ? dayjs(props.rangeTo) : props.firstOfMonth.endOf("month").startOf("D"),
);
const rangeLength = computed(() => rangeEnd.value.diff(rangeStart.value, "day") + 1);

const daysOfMonth = computed(() => {
	const daysOfMonth = [];
	for (let i = 0; i < rangeLength.value; i++) {
		const date = rangeStart.value.add(i, "day");
		daysOfMonth.push({
			dayName: date.format("ddd"),
			fullDayName: date.format("dddd"),
			date: date.format("YYYY-MM-DD"),
		});
	}

	if (props.dayFilter === "week") {
		// One column per weekday, showing the next upcoming one (or the last of the month)
		const start = dayjs().isAfter(rangeStart.value) ? dayjs().startOf("D") : rangeStart.value;
		return [1, 2, 3, 4, 5, 6, 0].map((dow) => {
			const matches = daysOfMonth.filter((d) => dayjs(d.date).day() === dow);
			const pick =
				matches.find((d) => !dayjs(d.date).isBefore(start)) || matches[matches.length - 1];
			return { ...pick, weekday: (dow + 6) % 7 };
		});
	}

	const [kind, value] = (props.dayFilter || "").split(":");
	if (kind === "dow") return daysOfMonth.filter((d) => dayjs(d.date).day() === Number(value));
	if (kind === "date") return daysOfMonth.filter((d) => d.date === value);
	return daysOfMonth;
});

const isWeekly = computed(() => props.dayFilter === "week");
const weekdayDialog = ref({ show: false, weekday: 0, name: "", employee: "" });

// Everyone currently listed (respecting the employee search box)
const visibleEmployees = computed(() =>
	props.employees
		.filter(
			(e) =>
				!employeeSearch.value?.length ||
				employeeSearch.value.some((item) => item.value === e.name),
		)
		.map((e) => e.name),
);

const dialogEmployeeLabels = computed(() => {
	const wanted = weekdayDialog.value.employee ? [weekdayDialog.value.employee] : visibleEmployees.value;
	return props.employees
		.filter((e) => wanted.includes(e.name))
		.map((e) => `${e.employee_name} (${e.name})`);
});

// From a header: everyone listed. From a cell: just that employee.
const openWeekday = (day: { weekday: number; fullDayName: string }, employee = "") => {
	weekdayDialog.value = { show: true, weekday: day.weekday, name: day.fullDayName, employee };
};

const employeeSearchOptions = computed(() => {
	return props.employees.map((employee: { name: string; employee_name: string }) => ({
		value: employee.name,
		label: `${employee.name}: ${employee.employee_name}`,
	}));
});

watch(
	() => [props.firstOfMonth, props.employeeFilters, props.shiftFilters, props.rangeFrom, props.rangeTo],
	() => {
		loading.value = true;
		events.fetch();
	},
	{ deep: true },
);

watch(loading, (val) => {
	if (!val) dropCell.value = { employee: "", date: "", shift: "" };
});

// Weekly view: for each employee and weekday, every distinct shift across the whole range,
// with how many days it covers (so "Fiid × 3" next to "Galab × 1" after a partial change).
const weeklyCells = computed(() => {
	const result: Record<string, Record<number, (Shift & { count: number })[]>> = {};
	if (!isWeekly.value || !events.data) return result;

	for (const employee of Object.keys(events.data)) {
		result[employee] = {};
		for (let i = 0; i < rangeLength.value; i++) {
			const date = rangeStart.value.add(i, "day");
			const weekday = (date.day() + 6) % 7;
			const shifts = events.data[employee]?.[date.format("YYYY-MM-DD")];
			if (!Array.isArray(shifts)) continue;

			const bucket = (result[employee][weekday] ||= []);
			for (const shift of shifts) {
				// Same shift type + status is one shift here; an empty location is not a different shift
				const found = bucket.find(
					(b) => b.shift_type === shift.shift_type && b.status === shift.status,
				);
				if (found) found.count++;
				else bucket.push({ ...shift, count: 1 });
			}
		}
	}
	return result;
});

const cell = (employee: string, day: { date: string; weekday?: number }) =>
	isWeekly.value
		? weeklyCells.value[employee]?.[day.weekday as number]
		: events.data?.[employee]?.[day.date];

const isHolidayOrLeave = (employee: string, day: string) =>
	events.data?.[employee]?.[day]?.holiday || events.data?.[employee]?.[day]?.leave;

const hasSameShift = (employee: string, day: string) =>
	Array.isArray(events.data?.[employee]?.[day]) &&
	events.data?.[employee]?.[day].some(
		(shift: Shift) =>
			shift.shift_type === hoveredCell.value.shift_type &&
			shift.shift_location === hoveredCell.value.shift_location &&
			shift.status === hoveredCell.value.shift_status,
	);

// RESOURCES

const events = createResource({
	url: "hrms.api.roster.get_events",
	auto: true,
	makeParams() {
		return {
			month_start: rangeStart.value.format("YYYY-MM-DD"),
			month_end: rangeStart.value.add(rangeLength.value - 1, "day").format("YYYY-MM-DD"),
			employee_filters: props.employeeFilters,
			shift_filters: props.shiftFilters,
		};
	},
	onSuccess() {
		loading.value = false;
	},
	onError(error: { messages: string[] }) {
		raiseToast("error", error.messages[0]);
	},
	transform: (data: Events) => {
		const mappedEvents: MappedEvents = {};
		for (const employee in data) {
			mapEventsToDates(data, mappedEvents, employee);
		}
		return mappedEvents;
	},
});
defineExpose({ events });

const swapShift = createResource({
	url: "hrms.api.roster.swap_shift",
	makeParams() {
		return {
			src_shift: hoveredCell.value.shift,
			src_date: hoveredCell.value.date,
			tgt_employee: dropCell.value.employee,
			tgt_date: dropCell.value.date,
			tgt_shift: dropCell.value.shift,
		};
	},
	onSuccess: () => {
		raiseToast("success", `Shift ${dropCell.value.shift ? "swapped" : "moved"} successfully!`);
		events.fetch();
	},
	onError(error: { messages: string[] }) {
		loading.value = false;
		raiseToast("error", error.messages[0]);
	},
});

const mapEventsToDates = (data: Events, mappedEvents: MappedEvents, employee: string) => {
	mappedEvents[employee] = {};
	for (let d = 0; d < rangeLength.value; d++) {
		const date = rangeStart.value.add(d, "day");
		const key = date.format("YYYY-MM-DD");

		for (const event of Object.values(data[employee])) {
			let result: Holiday | Leave | undefined;
			if ("holiday" in event) {
				result = handleHoliday(event, date);
				if (result) {
					mappedEvents[employee][key] = result;
					break;
				}
			} else if ("leave" in event) {
				result = handleLeave(event, date);
				if (result) {
					mappedEvents[employee][key] = result;
					break;
				}
			} else handleShifts(event, date, mappedEvents, employee, key);
		}
		sortShiftsByStartTime(mappedEvents, employee, key);
	}
};

const handleHoliday = (event: HolidayWithDate, date: Dayjs) => {
	if (date.isSame(event.holiday_date)) {
		return {
			holiday: event.holiday,
			description: event.description,
			weekly_off: event.weekly_off,
		};
	}
};

const handleLeave = (event: LeaveApplication, date: Dayjs) => {
	if (dayjs(event.from_date).isSameOrBefore(date) && dayjs(event.to_date).isSameOrAfter(date))
		return {
			leave: event.leave,
			leave_type: event.leave_type,
		};
};

const handleShifts = (
	event: ShiftAssignment,
	date: Dayjs,
	mappedEvents: MappedEvents,
	employee: string,
	key: string,
) => {
	if (
		dayjs(event.start_date).isSameOrBefore(date) &&
		(dayjs(event.end_date).isSameOrAfter(date) || !event.end_date)
	) {
		if (!Array.isArray(mappedEvents[employee][key])) mappedEvents[employee][key] = [];
		mappedEvents[employee][key].push({
			name: event.name,
			shift_type: event.shift_type,
			shift_location: event.shift_location,
			status: event.status,
			start_time: dayjs(event.start_time, "hh:mm:ss").format("HH:mm"),
			end_time: dayjs(event.end_time, "hh:mm:ss").format("HH:mm"),
			color: event.color.toLowerCase() as Color,
		});
	}
};

const sortShiftsByStartTime = (mappedEvents: MappedEvents, employee: string, key: string) => {
	if (Array.isArray(mappedEvents[employee][key]))
		mappedEvents[employee][key].sort((a: Shift, b: Shift) =>
			a.start_time.localeCompare(b.start_time),
		);
};
</script>

<style>
th,
td {
	@apply max-w-36 min-w-36;
	font-size: 0.875rem;
}

th:first-child,
td:first-child {
	@apply sticky left-0 max-w-64 min-w-64 bg-white border-r;
}

.blocked-cell {
	@apply text-sm text-gray-500 text-center p-2;
}
</style>
