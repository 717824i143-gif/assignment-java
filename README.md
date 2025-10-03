package assignment;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.*;

public class PharmacyConsoleApp {
    // Simple in-memory repositories
    private static final List<Medicine> medicines = new ArrayList<>();
    private static final List<Customer> customers = new ArrayList<>();
    private static final List<Prescription> prescriptions = new ArrayList<>();
    private static final List<Pharmacist> pharmacists = new ArrayList<>();
    private static final List<Sale> sales = new ArrayList<>();

    private static final Scanner sc = new Scanner(System.in);
    private static final DateTimeFormatter DF = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    public static void main(String[] args) {
        seedDemoData();
        while (true) {
            System.out.println("Pharmacy Management - Menu");
            System.out.println("1. Add Medicine");
            System.out.println("2. Add Customer");
            System.out.println("3. Add Prescription");
            System.out.println("4. Make Sale");
            System.out.println("5. Display Medicines");
            System.out.println("6. Display Prescriptions");
            System.out.println("7. Exit");
            System.out.print("Choose: ");
            String choice = sc.nextLine().trim();
            switch (choice) {
                case "1": addMedicine(); break;
                case "2": addCustomer(); break;
                case "3": addPrescription(); break;
                case "4": makeSale(); break;
                case "5": displayMedicines(); break;
                case "6": displayPrescriptions(); break;
                case "7": System.out.println("Exiting. Bye."); return;
                default: System.out.println("Invalid choice.");
            }
        }
    }

    private static void seedDemoData() {
        pharmacists.add(new Pharmacist("P1", "Dr. Mehta"));
        pharmacists.add(new Pharmacist("P2", "Mr. Sharma"));

        medicines.add(new Medicine("M001", "Paracetamol", false, 100, LocalDate.parse("2026-12-31", DF), 20.0));
        medicines.add(new Medicine("M002", "Amoxicillin", true, 50, LocalDate.parse("2025-05-15", DF), 45.0));
        medicines.add(new Medicine("M003", "Cough Syrup", false, 30, LocalDate.parse("2024-08-01", DF), 75.0));

        customers.add(new Customer("C001", "Ravi"));
        customers.add(new Customer("C002", "Anita"));
    }

    private static void addMedicine() {
        System.out.print("Medicine ID: ");
        String id = sc.nextLine().trim();
        System.out.print("Name: ");
        String name = sc.nextLine().trim();
        System.out.print("Prescription required? (y/n): ");
        boolean presReq = sc.nextLine().trim().equalsIgnoreCase("y");
        System.out.print("Quantity in stock: ");
        int qty = Integer.parseInt(sc.nextLine().trim());
        System.out.print("Expiry date (yyyy-MM-dd): ");
        LocalDate exp = LocalDate.parse(sc.nextLine().trim(), DF);
        System.out.print("Unit price: ");
        double price = Double.parseDouble(sc.nextLine().trim());
        medicines.add(new Medicine(id, name, presReq, qty, exp, price));
        System.out.println("Medicine added.");
    }

    private static void addCustomer() {
        System.out.print("Customer ID: ");
        String id = sc.nextLine().trim();
        System.out.print("Name: ");
        String name = sc.nextLine().trim();
        customers.add(new Customer(id, name));
        System.out.println("Customer added.");
    }

    private static void addPrescription() {
        System.out.print("Prescription ID: ");
        String pid = sc.nextLine().trim();
        System.out.print("Customer ID: ");
        String cid = sc.nextLine().trim();
        Customer c = findCustomer(cid);
        if (c == null) { System.out.println("Customer not found."); return; }
        Prescription p = new Prescription(pid, c);
        while (true) {
            System.out.print("Add item? (y/n): ");
            if (!sc.nextLine().trim().equalsIgnoreCase("y")) break;
            System.out.print("Medicine ID: ");
            String mid = sc.nextLine().trim();
            Medicine m = findMedicine(mid);
            if (m == null) { System.out.println("Medicine not found."); continue; }
            System.out.print("Quantity prescribed: ");
            int q = Integer.parseInt(sc.nextLine().trim());
            p.addItem(new PrescriptionItem(UUID.randomUUID().toString(), m, q));
        }
        prescriptions.add(p);
        System.out.println("Prescription added.");
    }

    private static void makeSale() {
        System.out.print("Sale ID: ");
        String sid = sc.nextLine().trim();
        System.out.print("Customer ID: ");
        String cid = sc.nextLine().trim();
        Customer c = findCustomer(cid);
        if (c == null) { System.out.println("Customer not found."); return; }

        System.out.println("Available pharmacists:");
        for (Pharmacist p : pharmacists) System.out.println(p);
        System.out.print("Pharmacist ID who handles sale: ");
        String pid = sc.nextLine().trim();
        Pharmacist pharm = findPharmacist(pid);
        if (pharm == null) { System.out.println("Pharmacist not found."); return; }

        Sale sale = new Sale(sid, c, pharm, LocalDate.now());

        while (true) {
            System.out.print("Add medicine to sale? (y/n): ");
            if (!sc.nextLine().trim().equalsIgnoreCase("y")) break;
            System.out.print("Medicine ID: ");
            String mid = sc.nextLine().trim();
            Medicine m = findMedicine(mid);
            if (m == null) { System.out.println("Medicine not found."); continue; }
            if (m.isExpired()) { System.out.println("Cannot sell: medicine expired on " + m.getExpiryDate()); continue; }
            System.out.print("Quantity: ");
            int qty = Integer.parseInt(sc.nextLine().trim());
            if (qty <= 0) { System.out.println("Invalid quantity"); continue; }
            if (qty > m.getQuantity()) { System.out.println("Insufficient stock. Available: " + m.getQuantity()); continue; }

            PrescriptionItem matchedPresItem = null;
            if (m.isPrescriptionRequired()) {
                System.out.print("This medicine requires prescription. Provide Prescription ID: ");
                String prid = sc.nextLine().trim();
                Prescription pres = findPrescription(prid);
                if (pres == null) { System.out.println("Prescription not found. Cannot sell."); continue; }
                // find matching item in prescription
                matchedPresItem = pres.findItemByMedicineId(mid);
                if (matchedPresItem == null) { System.out.println("Prescription does not contain this medicine. Cannot sell."); continue; }
                if (matchedPresItem.getQuantity() < qty) { System.out.println("Prescription quantity insufficient. Prescribed: " + matchedPresItem.getQuantity()); continue; }
            }

            // All checks passed; create SaleDetail
            SaleDetail sd = new SaleDetail(UUID.randomUUID().toString(), m, qty, m.getUnitPrice(), matchedPresItem);
            sale.addDetail(sd);
            // reduce stock immediately
            m.decreaseQuantity(qty);
            System.out.println("Added to sale. Current medicine stock: " + m.getQuantity());
            if (m.getExpiryDate().isBefore(LocalDate.now().plusMonths(3))) {
                System.out.println("Warning: medicine expires soon on " + m.getExpiryDate());
            }
        }

        if (sale.getDetails().isEmpty()) { System.out.println("No items sold. Sale aborted."); return; }

        // finalize sale, print bill
        sales.add(sale);
        sale.printBill();
    }

    private static void displayMedicines() {
        System.out.println("Medicines:");
        for (Medicine m : medicines) {
            System.out.println(m.detailedString());
        }
    }

    private static void displayPrescriptions() {
        System.out.println("Prescriptions:");
        for (Prescription p : prescriptions) System.out.println(p);
    }

    // find helpers
    private static Medicine findMedicine(String id) {
        for (Medicine m : medicines) if (m.getId().equalsIgnoreCase(id)) return m; return null;
    }
    private static Customer findCustomer(String id) {
        for (Customer c : customers) if (c.getId().equalsIgnoreCase(id)) return c; return null;
    }
    private static Prescription findPrescription(String id) {
        for (Prescription p : prescriptions) if (p.getId().equalsIgnoreCase(id)) return p; return null;
    }
    private static Pharmacist findPharmacist(String id) {
        for (Pharmacist p : pharmacists) if (p.getId().equalsIgnoreCase(id)) return p; return null;
    }
}

// -------------------- Domain classes --------------------
class Medicine {
    private final String id;
    private final String name;
    private final boolean prescriptionRequired;
    private int quantity; // total units available
    private final LocalDate expiryDate; // simplified: single expiry date (real systems have batches)
    private final double unitPrice;

    public Medicine(String id, String name, boolean prescriptionRequired, int quantity, LocalDate expiryDate, double unitPrice) {
        this.id = id; this.name = name; this.prescriptionRequired = prescriptionRequired; this.quantity = quantity; this.expiryDate = expiryDate; this.unitPrice = unitPrice;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public boolean isPrescriptionRequired() { return prescriptionRequired; }
    public int getQuantity() { return quantity; }
    public LocalDate getExpiryDate() { return expiryDate; }
    public double getUnitPrice() { return unitPrice; }

    public boolean isExpired() { return expiryDate.isBefore(LocalDate.now()) || expiryDate.isEqual(LocalDate.now().minusDays(1)); }

    public void decreaseQuantity(int q) { if (q <= quantity) quantity -= q; else throw new IllegalArgumentException("Insufficient stock"); }

    public String detailedString() {
        return String.format("%s - %s | PresReq: %s | Qty: %d | Expiry: %s | Price: %.2f",
                id, name, prescriptionRequired ? "Yes" : "No", quantity, expiryDate.toString(), unitPrice);
    }

    @Override public String toString() { return id + ": " + name; }
}

class Customer {
    private final String id;
    private final String name;
    public Customer(String id, String name) { this.id = id; this.name = name; }
    public String getId() { return id; }
    public String getName() { return name; }
    @Override public String toString() { return id + " - " + name; }
}

class Prescription {
    private final String id;
    private final Customer customer;
    private final LocalDate dateIssued;
    private final List<PrescriptionItem> items = new ArrayList<>();

    public Prescription(String id, Customer customer) { this.id = id; this.customer = customer; this.dateIssued = LocalDate.now(); }
    public String getId() { return id; }
    public Customer getCustomer() { return customer; }
    public void addItem(PrescriptionItem it) { items.add(it); }
    public PrescriptionItem findItemByMedicineId(String mid) {
        for (PrescriptionItem i : items) if (i.getMedicine().getId().equalsIgnoreCase(mid)) return i; return null;
    }
    @Override public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Prescription ").append(id).append(" for ").append(customer.getName()).append(" on ").append(dateIssued).append("\n");
        for (PrescriptionItem it : items) sb.append("  ").append(it).append("\n");
        return sb.toString();
    }
}

class PrescriptionItem {
    private final String id;
    private final Medicine medicine;
    private final int quantity;

    public PrescriptionItem(String id, Medicine medicine, int quantity) { this.id = id; this.medicine = medicine; this.quantity = quantity; }
    public String getId() { return id; }
    public Medicine getMedicine() { return medicine; }
    public int getQuantity() { return quantity; }
    @Override public String toString() { return String.format("%s x %d (med: %s)", medicine.getName(), quantity, medicine.getId()); }
}

class Pharmacist {
    private final String id;
    private final String name;
    public Pharmacist(String id, String name) { this.id = id; this.name = name; }
    public String getId() { return id; }
    public String getName() { return name; }
    @Override public String toString() { return id + " - " + name; }
}

class Sale {
    private final String id;
    private final Customer customer;
    private final Pharmacist pharmacist;
    private final LocalDate date;
    private final List<SaleDetail> details = new ArrayList<>();

    public Sale(String id, Customer customer, Pharmacist pharmacist, LocalDate date) {
        this.id = id; this.customer = customer; this.pharmacist = pharmacist; this.date = date;
    }

    public void addDetail(SaleDetail d) { details.add(d); }
    public List<SaleDetail> getDetails() { return details; }

    public void printBill() {
        System.out.println("\n---- SALE BILL ----");
        System.out.println("Sale ID: " + id);
        System.out.println("Date: " + date);
        System.out.println("Customer: " + customer);
        System.out.println("Pharmacist: " + pharmacist);
        System.out.println("\nItems:");
        double total = 0.0;
        for (SaleDetail sd : details) {
            double line = sd.getQuantity() * sd.getUnitPrice();
            total += line;
            String regulated = sd.getMedicine().isPrescriptionRequired() ? "[REGULATED]" : "";
            String presRef = sd.getPrescriptionItem()!=null ? "(Prescribed qty: " + sd.getPrescriptionItem().getQuantity() + ")" : "";
            String expired = sd.getMedicine().getExpiryDate().isBefore(LocalDate.now()) ? "[EXPIRED]" : "";
            String nearExpiry = "";
            if (!sd.getMedicine().getExpiryDate().isBefore(LocalDate.now()) && sd.getMedicine().getExpiryDate().isBefore(LocalDate.now().plusMonths(3))) nearExpiry = "[EXPIRY_SOON]";
            System.out.printf("%s | %s x %d @ %.2f = %.2f %s %s %s%n", sd.getMedicine().getId(), sd.getMedicine().getName(), sd.getQuantity(), sd.getUnitPrice(), line, regulated, presRef, (expired + nearExpiry).trim());
        }
        System.out.printf("TOTAL: %.2f%n", total);
        System.out.println("-------------------\n");
    }
}

class SaleDetail {
    private final String id;
    private final Medicine medicine;
    private final int quantity;
    private final double unitPrice;
    private final PrescriptionItem prescriptionItem; // may be null for OTC

    public SaleDetail(String id, Medicine medicine, int quantity, double unitPrice, PrescriptionItem prescriptionItem) {
        this.id = id; this.medicine = medicine; this.quantity = quantity; this.unitPrice = unitPrice; this.prescriptionItem = prescriptionItem;
    }

    public String getId() { return id; }
    public Medicine getMedicine() { return medicine; }
    public int getQuantity() { return quantity; }
    public double getUnitPrice() { return unitPrice; }
    public PrescriptionItem getPrescriptionItem() { return prescriptionItem; }
}

