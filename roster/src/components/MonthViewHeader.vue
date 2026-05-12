<template>
	<div class="flex items-center">
		<!-- Month Change -->
		<div class="flex items-center bg-gray-50 rounded-md space-x-0.5">
			<Button icon="chevron-left" variant="ghost" @click="emit('addToMonth', -1)" />
			<span class="w-32 text-center font-medium text-base">
				{{ props.firstOfMonth.format("MMMM") }}, {{ firstOfMonth.format("YYYY") }}
			</span>
			<Button icon="chevron-right" variant="ghost" @click="emit('addToMonth', 1)" />
		</div>

		<!-- Filters -->
		<div class="ml-auto space-x-2.5 flex">
			<div v-for="[key, value] of Object.entries(filters)" :key="key" class="w-40">
				<FormControl
					type="autocomplete"
					:placeholder="filterPlaceholder(key as FilterField)"
					:options="value.options"
					v-model="value.model"
					:disabled="!value.options.length"
				/>
			</div>
			<Button icon="x" @click="Object.values(filters).forEach((d) => (d.model = null))" />
		</div>
	</div>
</template>

<script setup lang="ts">
import { reactive, watch } from "vue";
import { FormControl, createResource, createListResource } from "frappe-ui";
import { Dayjs } from "dayjs";

import { raiseToast } from "../utils";

export type FilterField =
	| "company"
	| "department"
	| "branch"
	| "designation"
	| "custom_postal_territory"
	| "custom_postal_profile"
	| "shift_type"
	| "shift_location";

const props = defineProps<{
	firstOfMonth: Dayjs;
}>();

const emit = defineEmits<{
	(e: "addToMonth", change: number): void;
	(e: "updateFilters", newFilters: { [K in FilterField]: string }): void;
}>();

type TerritoryOptionRow = { name: string; postal_territory?: string };

const filters: {
	[K in FilterField]: {
		options: (string | { label: string; value: string })[];
		model?: { value: string } | null;
	};
} = reactive({
	company: { options: [], model: null },
	department: { options: [], model: null },
	branch: { options: [], model: null },
	designation: { options: [], model: null },
	custom_postal_territory: { options: [], model: null },
	custom_postal_profile: { options: [], model: null },
	shift_type: { options: [], model: null },
	shift_location: { options: [], model: null },
});

watch(
	() => filters.company.model,
	(val) => {
		if (val?.value) {
			getFilterOptions("department", { company: val.value });
			getFilterOptions("custom_postal_territory", { company: val.value });
			getFilterOptions("custom_postal_profile", { company: val.value });
		} else {
			filters.department.model = null;
			filters.department.options = [];
			filters.custom_postal_territory.model = null;
			filters.custom_postal_territory.options = [];
			filters.custom_postal_profile.model = null;
			filters.custom_postal_profile.options = [];
		}
	},
);

watch(filters, (val) => {
	const newFilters = {
		company: val.company.model?.value || "",
		department: val.department.model?.value || "",
		branch: val.branch.model?.value || "",
		designation: val.designation.model?.value || "",
		custom_postal_territory: val.custom_postal_territory.model?.value || "",
		custom_postal_profile: val.custom_postal_profile.model?.value || "",
		shift_type: val.shift_type.model?.value || "",
		shift_location: val.shift_location.model?.value || "",
	};
	emit("updateFilters", newFilters);
});

const toTitleCase = (str: string) =>
	str
		.split("_")
		.map((s) => s.charAt(0).toUpperCase() + s.slice(1))
		.join(" ");

const filterPlaceholder = (field: FilterField): string => {
	switch (field) {
		case "custom_postal_territory":
			return "Postal Territory";
		case "custom_postal_profile":
			return "POS Profile";
		default:
			return toTitleCase(field);
	}
};

const optionsDoctype = (field: FilterField): string => {
	switch (field) {
		case "custom_postal_territory":
			return "Postal Territory";
		case "custom_postal_profile":
			return "POS Profile";
		default:
			return toTitleCase(field);
	}
};

// RESOURCES

const defaultCompany = createResource({
	url: "hrms.api.roster.get_default_company",
	auto: true,
	onSuccess: () => {
		["company", "branch", "designation", "shift_type", "shift_location"].forEach((field) =>
			getFilterOptions(field as FilterField),
		);
	},
});

const territoryFilterOptions = (rows: TerritoryOptionRow[]) =>
	rows.map((item) => ({
		label: item.postal_territory?.trim() || item.name,
		value: item.name,
	}));

const getFilterOptions = (field: FilterField, listFilters: { company?: string } = {}) => {
	const isPostalTerritory = field === "custom_postal_territory";
	createListResource({
		doctype: optionsDoctype(field),
		fields: isPostalTerritory ? ["name", "postal_territory"] : ["name"],
		filters: listFilters,
		pageLength: 100,
		auto: true,
		onSuccess: (data: TerritoryOptionRow[]) => {
			const value = field === "company" ? defaultCompany.data : "";
			filters[field].model = { value };
			filters[field].options = isPostalTerritory ? territoryFilterOptions(data) : data.map((item) => item.name);
		},
		onError(error: { messages: string[] }) {
			raiseToast("error", error.messages[0]);
		},
	});
};
</script>
