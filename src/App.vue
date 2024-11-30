<template>
    <ul class="nav nav-tabs">
        <li class="nav-item" v-for="tab in tabs" :key="tab.key">
            <a :class="{ 'nav-link': true, active: tab.key == selected_tab }" aria-current="page" @click="changeTab(tab.key)">
                {{ tab.label }}
            </a>
        </li>
    </ul>
    <template v-if="selected_tab == 'basic'">
        <h2 class="display-4 text-center">Quality recycling calculator</h2>
        <p class="text-center">
            This simulator helps to choose which modules to use depending on the available module slots & base productivity (machine/recipe). Input
            value represents how many times an item is to be crafted. Every cycle, the non-legendary crafts from the previous cycle are recycled and
            reused to redo a quality craft
            <br />
            Note: It is implied that we are using modules of highest tier & quality, and targeting legendary output only
        </p>
    </template>
    <template v-if="selected_tab == 'holmium'">
        <h2 class="display-4 text-center">Holmium plate quality upcrafting calculator</h2>
        <p class="text-center">
            This calculator assumes the input of regular Holmium plates, and simulates probabilites of a crafting chain Plates -> Superconductor ->
            Supercapacitor -> Recycling, and outputs the statistical distribution of received holmium plates (producing exactly enough superconductors
            to craft the supercapacitor of corresponding quality).<br />You can simulate the modules used for both crafts (Legendary craft always
            assumes full productivity)
            <br />
            Note: Legendary plates outputs are never used to craft the capacitor, and are added with each cycle
        </p>
    </template>
    <main>
        <div class="container mt-3">
            <div class="row">
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'basic'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Input </span>
                    </div>
                    <input class="form-control" type="number" v-model="input" /><br />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'basic'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Machine module slots </span>
                    </div>
                    <input type="number" class="form-control" max="5" min="0" v-model="basic_total_modules" />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'basic'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1">
                            Base productivity (due to the machine/tech level) %. Ex: '50' if using EMP
                        </span>
                    </div>
                    <input type="number" class="form-control" max="5" min="0" v-model="basic_base_prod" />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'basic'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1">
                            Productivity modules (legendary T3 - the rest will be assumed quality T3)
                        </span>
                    </div>
                    <input type="number" class="form-control" max="5" min="0" v-model="basic_prod_module_count" />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'holmium'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Holmium plate input </span>
                    </div>
                    <input class="form-control" type="number" v-model="input" /><br />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'holmium'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Superconductor quality modules /5 (remainder will be prod) </span>
                    </div>
                    <input type="number" class="form-control" max="5" min="0" v-model="supercond_q_modules" />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex" v-if="selected_tab == 'holmium'">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Supercapacitor quality modules /5 (remainder will be prod) </span>
                    </div>
                    <input class="form-control" type="number" max="5" min="0" v-model="supercapa_q_modules" />
                </div>
                <div class="input-group mb-3 col-6 align-self-start d-flex">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1"> Cycles </span>
                    </div>
                    <input class="form-control" type="number" v-model="cycles" /><br />
                </div>
                <input class="btn btn-primary" type="button" @click="compute()" value="Simulate !" />
            </div>
        </div>

        <hr />
        <table class="table">
            <thead>
                <tr>
                    <th></th>
                    <th>Normal</th>
                    <th>Uncommon</th>
                    <th>Rare</th>
                    <th>Epic</th>
                    <th>Legendary</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(result, idx) in results" :key="idx">
                    <th>After {{ idx + 1 }} cycle<template v-if="idx > 0">s</template></th>
                    <td>{{ round(result[0], 2) }}</td>
                    <td>{{ round(result[1], 2) }}</td>
                    <td>{{ round(result[2], 2) }}</td>
                    <td>{{ round(result[3], 2) }}</td>
                    <td>{{ round(result[4], 2) }}</td>
                </tr>
            </tbody>
        </table>
    </main>
</template>

<script>
export default {
    data() {
        return {
            tabs: [
                { key: 'basic', label: 'Single product recycling' },
                { key: 'holmium', label: 'Holmium plates' },
            ],
            selected_tab: 'basic',
            // Input shared
            input: 1000,
            cycles: 20,
            // Input basic
            basic_total_modules: 4,
            basic_base_prod: 0,
            basic_prod_module_count: 2,
            // Input holmium
            supercond_q_modules: 0,
            supercapa_q_modules: 0,
            // Output
            results: [],
            // Inner
            last_cycle_inputs: [],
            last_cycle_results: [],
        }
    },
    mounted() {
        this.reset()
    },
    computed: {
        basic_prod() {
            return [
                1 + this.basic_base_prod / 100 + this.basic_prod_module_count * 0.25,
                1 + this.basic_base_prod / 100 + this.basic_prod_module_count * 0.25,
                1 + this.basic_base_prod / 100 + this.basic_prod_module_count * 0.25,
                1 + this.basic_base_prod / 100 + this.basic_prod_module_count * 0.25,
                1 + this.basic_base_prod / 100 + this.basic_total_modules * 0.25,
            ]
        },
        basic_qual_chance() {
            var qual_chance = (this.basic_total_modules - this.basic_prod_module_count) * 0.062
            return [1 - qual_chance, qual_chance * 0.9, qual_chance * 0.09, qual_chance * 0.009, qual_chance * 0.001]
        },
        cond_prod() {
            // Per quality craft
            return [
                1.5 + (5 - this.supercond_q_modules) * 0.25,
                1.5 + (5 - this.supercond_q_modules) * 0.25,
                1.5 + (5 - this.supercond_q_modules) * 0.25,
                1.5 + (5 - this.supercond_q_modules) * 0.25,
                1.5 + 5 * 0.25, // Always prod
            ]
        },
        capa_prod() {
            return [
                1.5 + (5 - this.supercapa_q_modules) * 0.25,
                1.5 + (5 - this.supercapa_q_modules) * 0.25,
                1.5 + (5 - this.supercapa_q_modules) * 0.25,
                1.5 + (5 - this.supercapa_q_modules) * 0.25,
                1.5 + 5 * 0.25, // Always prod
            ]
        },
        cond_qual_chance() {
            // For each level. 0 : same qual, 1 : next, etc
            var qual_chance = this.supercond_q_modules * 0.062
            return [1 - qual_chance, qual_chance * 0.9, qual_chance * 0.09, qual_chance * 0.009, qual_chance * 0.001]
        },
        capa_qual_chance() {
            // For each level. 0 : same qual, 1 : next, etc
            var qual_chance = this.supercapa_q_modules * 0.062
            return [1 - qual_chance, qual_chance * 0.9, qual_chance * 0.09, qual_chance * 0.009, qual_chance * 0.001]
        },
        recycler_qual_chance() {
            // For each level. 0 : same qual, 1 : next, etc
            var qual_chance = 4 * 0.062
            return [1 - qual_chance, qual_chance * 0.9, qual_chance * 0.09, qual_chance * 0.009, qual_chance * 0.001]
        },
        capa_plate_ratio() {
            // Per quality craft
            // 0-1 ratio of holmium plates to be spent on capacitors. 1 needed for 2 cond, and 2 needed for 1 capa. Assuming cond_prod, this brings us to this formula:
            return [
                (2 * this.cond_prod[0]) / (2 * this.cond_prod[0] + 1),
                (2 * this.cond_prod[1]) / (2 * this.cond_prod[1] + 1),
                (2 * this.cond_prod[2]) / (2 * this.cond_prod[2] + 1),
                (2 * this.cond_prod[3]) / (2 * this.cond_prod[3] + 1),
                (2 * 2.75) / (2 * 2.75 + 1), // Max cond prod
            ]
        },
    },
    methods: {
        changeTab(key) {
            this.selected_tab = key
            this.reset()
        },
        round(num, decimals) {
            var p = Math.pow(10, decimals)
            return Math.round(num * p) / p
        },
        reset() {
            this.out = [0, 0, 0, 0, 0]
            this.last_cycle_inputs = [0, 0, 0, 0, 0]
            this.last_cycle_results = [0, 0, 0, 0, 0]
            this.results = []
        },
        compute() {
            this.reset()
            if (this.selected_tab == 'basic') {
                this.compute_basic()
            } else {
                this.compute_holmium()
            }
        },
        compute_basic() {
            this.last_cycle_inputs = [this.input, 0, 0, 0, 0]
            this.last_cycle_results = [0, 0, 0, 0, 0]
            for (var i = 0; i < this.cycles; i++) {
                this.compute_single_cycle(this.basic_prod, this.basic_qual_chance, [1, 1, 1, 1, 1], true)
            }
        },
        compute_holmium() {
            this.last_cycle_inputs = [this.input, 0, 0, 0, 0]
            this.last_cycle_results = [0, 0, 0, 0, 0]
            for (var i = 0; i < this.cycles; i++) {
                this.compute_single_cycle(
                    this.capa_prod,
                    this.capa_qual_chance,
                    this.capa_plate_ratio.map((a) => a / 2),
                    false
                )
            }
        },
        // While the table can be directly used for convertion from normal quality, when converting from uncommon and above, the overflowing values of the table have to be added
        calculateQualChance(from = 0, to = 0, table) {
            if (to == 4 && from > 0) {
                // Addition only happens when going to legendary from non common
                if (from == 1) {
                    return table[3] + table[4]
                } else if (from == 2) {
                    return table[2] + table[3] + table[4]
                } else if (from == 3) {
                    return table[1] + table[2] + table[3] + table[4]
                } else if (from == 4) {
                    return 1
                }
            } else {
                return table[to - from]
            }
        },
        // multiplier factor is a static multiplier on the craft. For example divide by 2 if 2 inputs are consumed for the desired output
        // All inputs are expected to be arrays of 5 items, each for every tier of quality
        // keep_product: whether the objective is the legendary product, or otherwise it will be recycled into legendary input (for the holmium plates path)
        compute_single_cycle(prod_factor_table, qual_chance_table, multiplier_factor_table, keep_product = true) {
            if (keep_product) {
                // If the goal is the product, then recycle at the beginning of each cycle (except legendary)
                this.last_cycle_inputs[0] += this.last_cycle_results[0] * 0.25 * this.calculateQualChance(0, 0, this.recycler_qual_chance)
                this.last_cycle_inputs[1] +=
                    this.last_cycle_results[0] * 0.25 * this.calculateQualChance(0, 1, this.recycler_qual_chance) +
                    this.last_cycle_results[1] * 0.25 * this.calculateQualChance(1, 1, this.recycler_qual_chance)
                this.last_cycle_inputs[2] +=
                    this.last_cycle_results[0] * 0.25 * this.calculateQualChance(0, 2, this.recycler_qual_chance) +
                    this.last_cycle_results[1] * 0.25 * this.calculateQualChance(1, 2, this.recycler_qual_chance) +
                    this.last_cycle_results[2] * 0.25 * this.calculateQualChance(2, 2, this.recycler_qual_chance)
                this.last_cycle_inputs[3] +=
                    this.last_cycle_results[0] * 0.25 * this.calculateQualChance(0, 3, this.recycler_qual_chance) +
                    this.last_cycle_results[1] * 0.25 * this.calculateQualChance(1, 3, this.recycler_qual_chance) +
                    this.last_cycle_results[2] * 0.25 * this.calculateQualChance(2, 3, this.recycler_qual_chance) +
                    this.last_cycle_results[3] * 0.25 * this.calculateQualChance(3, 3, this.recycler_qual_chance)
                this.last_cycle_inputs[4] +=
                    this.last_cycle_results[0] * 0.25 * this.calculateQualChance(0, 4, this.recycler_qual_chance) +
                    this.last_cycle_results[1] * 0.25 * this.calculateQualChance(1, 4, this.recycler_qual_chance) +
                    this.last_cycle_results[2] * 0.25 * this.calculateQualChance(2, 4, this.recycler_qual_chance) +
                    this.last_cycle_results[3] * 0.25 * this.calculateQualChance(3, 4, this.recycler_qual_chance)
                // Not recycling last cycle's legendary results
            }

            // Now use all inputs to craft
            var craft_results = []
            craft_results[0] =
                this.last_cycle_inputs[0] * multiplier_factor_table[0] * prod_factor_table[0] * this.calculateQualChance(0, 0, qual_chance_table)
            craft_results[1] =
                this.last_cycle_inputs[0] * multiplier_factor_table[0] * prod_factor_table[0] * this.calculateQualChance(0, 1, qual_chance_table) +
                this.last_cycle_inputs[1] * multiplier_factor_table[1] * prod_factor_table[1] * this.calculateQualChance(1, 1, qual_chance_table)
            craft_results[2] =
                this.last_cycle_inputs[0] * multiplier_factor_table[0] * prod_factor_table[0] * this.calculateQualChance(0, 2, qual_chance_table) +
                this.last_cycle_inputs[1] * multiplier_factor_table[1] * prod_factor_table[1] * this.calculateQualChance(1, 2, qual_chance_table) +
                this.last_cycle_inputs[2] * multiplier_factor_table[2] * prod_factor_table[2] * this.calculateQualChance(2, 2, qual_chance_table)
            craft_results[3] =
                this.last_cycle_inputs[0] * multiplier_factor_table[0] * prod_factor_table[0] * this.calculateQualChance(0, 3, qual_chance_table) +
                this.last_cycle_inputs[1] * multiplier_factor_table[1] * prod_factor_table[1] * this.calculateQualChance(1, 3, qual_chance_table) +
                this.last_cycle_inputs[2] * multiplier_factor_table[2] * prod_factor_table[2] * this.calculateQualChance(2, 3, qual_chance_table) +
                this.last_cycle_inputs[3] * multiplier_factor_table[3] * prod_factor_table[3] * this.calculateQualChance(3, 3, qual_chance_table)
            craft_results[4] =
                this.last_cycle_results[4] +
                this.last_cycle_inputs[0] * multiplier_factor_table[0] * prod_factor_table[0] * this.calculateQualChance(0, 4, qual_chance_table) +
                this.last_cycle_inputs[1] * multiplier_factor_table[1] * prod_factor_table[1] * this.calculateQualChance(1, 4, qual_chance_table) +
                this.last_cycle_inputs[2] * multiplier_factor_table[2] * prod_factor_table[2] * this.calculateQualChance(2, 4, qual_chance_table) +
                this.last_cycle_inputs[3] * multiplier_factor_table[3] * prod_factor_table[3] * this.calculateQualChance(3, 4, qual_chance_table)
            if (keep_product) {
                // Only if the goal is the product then use up legendary inputs for the craft
                craft_results[4] += this.last_cycle_inputs[4] * multiplier_factor_table[4] * prod_factor_table[4] // Legendary craft : no qual chance, and make sure that prod_factor_table[4] uses full prod
            }

            if (keep_product) {
                this.results.push(craft_results)
                this.last_cycle_inputs = [0, 0, 0, 0, 0] // All inputs have been used up
                this.last_cycle_results = craft_results
            } else {
                // If the quality upping of the inputs is the goal, then recycle at the end of the cycle
                this.last_cycle_inputs[0] = craft_results[0] * 0.25 * this.calculateQualChance(0, 0, this.recycler_qual_chance)
                this.last_cycle_inputs[1] =
                    craft_results[0] * 0.25 * this.calculateQualChance(0, 1, this.recycler_qual_chance) +
                    craft_results[1] * 0.25 * this.calculateQualChance(1, 1, this.recycler_qual_chance)
                this.last_cycle_inputs[2] =
                    craft_results[0] * 0.25 * this.calculateQualChance(0, 2, this.recycler_qual_chance) +
                    craft_results[1] * 0.25 * this.calculateQualChance(1, 2, this.recycler_qual_chance) +
                    craft_results[2] * 0.25 * this.calculateQualChance(2, 2, this.recycler_qual_chance)
                this.last_cycle_inputs[3] =
                    craft_results[0] * 0.25 * this.calculateQualChance(0, 3, this.recycler_qual_chance) +
                    craft_results[1] * 0.25 * this.calculateQualChance(1, 3, this.recycler_qual_chance) +
                    craft_results[2] * 0.25 * this.calculateQualChance(2, 3, this.recycler_qual_chance) +
                    craft_results[3] * 0.25 * this.calculateQualChance(3, 3, this.recycler_qual_chance)
                this.last_cycle_inputs[4] += // Add to the previous amount of legendary inputs
                    craft_results[0] * 0.25 * this.calculateQualChance(0, 4, this.recycler_qual_chance) +
                    craft_results[1] * 0.25 * this.calculateQualChance(1, 4, this.recycler_qual_chance) +
                    craft_results[2] * 0.25 * this.calculateQualChance(2, 4, this.recycler_qual_chance) +
                    craft_results[3] * 0.25 * this.calculateQualChance(3, 4, this.recycler_qual_chance) +
                    craft_results[4] * 0.25
                this.results.push([...this.last_cycle_inputs]) // If keeping inputs, then what we have after recycling crafts is this cycle's result
            }
        },
    },
}
</script>

<style scoped></style>
