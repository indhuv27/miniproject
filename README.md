# miniproject
/*
VehicleTracker.java

A simple Java Swing application that simulates tracking public transport vehicles using
simulated GPS coordinates.

Features:
 - Simulates multiple vehicles moving within a bounding box (latitude/longitude).
 - Shows vehicles on a simple map panel (dots) and in a table with their data.
 - Start / Stop simulation, Add / Remove vehicle, Adjust update interval.
 - Single-file: compile with `javac VehicleTracker.java` and run `java VehicleTracker`.

Notes:
 - This is a simulation; there is no real map tile background. The map panel draws a grid
   and scales lat/lon into the panel area.
 - For a production system, you'd integrate a map SDK (Google Maps, Leaflet via JxBrowser,
   or other mapping libraries) and fetch real GPS positions from a server.
*/

import javax.swing.*;
import javax.swing.event.*;
import javax.swing.table.*;
import java.awt.*;
import java.awt.event.*;
import java.util.*;
import java.util.concurrent.*;

public class VehicleTracker {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new TrackerFrame().setVisible(true));
    }
}

// Simple data model for a vehicle
class Vehicle {
    final String id;
    double lat;  // latitude
    double lon;  // longitude
    double speed; // meters per second
    double heading; // degrees 0-360
    long lastUpdated;

    Vehicle(String id, double lat, double lon) {
        this.id = id;
        this.lat = lat;
        this.lon = lon;
        this.speed = 5 + Math.random() * 10; // random speed
        this.heading = Math.random() * 360;
        this.lastUpdated = System.currentTimeMillis();
    }
}

// Main application frame
class TrackerFrame extends JFrame {
    private final MapPanel mapPanel;
    private final DefaultTableModel tableModel;
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
    private ScheduledFuture<?> scheduledTask;
    private final java.util.List<Vehicle> vehicles = Collections.synchronizedList(new ArrayList<>());

    // Bounding box for the simulated area (latitude/longitude)
    private final double minLat = 12.90;
    private final double maxLat = 13.05;
    private final double minLon = 77.50;
    private final double maxLon = 77.80;

    TrackerFrame() {
        super("Public Transport Vehicle Tracker - Simulated GPS");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(1000, 700);
        setLocationRelativeTo(null);

        mapPanel = new MapPanel(vehicles, minLat, maxLat, minLon, maxLon);

        // Table to show vehicles
        tableModel = new DefaultTableModel(new Object[]{"ID", "Latitude", "Longitude", "Speed (m/s)", "Heading"}, 0) {
            @Override
            public boolean isCellEditable(int row, int column) { return false; }
        };
        JTable table = new JTable(tableModel);
        table.setFillsViewportHeight(true);
        JScrollPane tableScroll = new JScrollPane(table);
        tableScroll.setPreferredSize(new Dimension(380, 300));

        // Controls
        JButton startBtn = new JButton("Start");
        JButton stopBtn = new JButton("Stop");
        JButton addBtn = new JButton("Add Vehicle");
        JButton removeBtn = new JButton("Remove Selected");
        JSlider intervalSlider = new JSlider(100, 2000, 500); // update interval in ms
        intervalSlider.setMajorTickSpacing(500);
        intervalSlider.setMinorTickSpacing(100);
        intervalSlider.setPaintTicks(true);
        intervalSlider.setPaintLabels(true);
        JLabel intervalLabel = new JLabel("Update interval: 500 ms");

        intervalSlider.addChangeListener(e -> intervalLabel.setText("Update interval: " + intervalSlider.getValue() + " ms"));

        startBtn.addActionListener(e -> startSimulation(intervalSlider.getValue()));
        stopBtn.addActionListener(e -> stopSimulation());

        addBtn.addActionListener(e -> {
            String id = "V" + (vehicles.size() + 1);
            double lat = minLat + Math.random() * (maxLat - minLat);
            double lon = minLon + Math.random() * (maxLon - minLon);
            Vehicle v = new Vehicle(id, lat, lon);
            vehicles.add(v);
            refreshTable();
            mapPanel.repaint();
        });

        removeBtn.addActionListener(e -> {
            int sel = table.getSelectedRow();
            if (sel >= 0) {
                String id = (String) tableModel.getValueAt(sel, 0);
                vehicles.removeIf(v -> v.id.equals(id));
                refreshTable();
                mapPanel.repaint();
            } else {
                JOptionPane.showMessageDialog(this, "Select a vehicle row to remove.", "No selection", JOptionPane.WARNING_MESSAGE);
            }
        });

        // Layout
        JPanel right = new JPanel(new BorderLayout(8, 8));
        JPanel controls = new JPanel(new GridLayout(0, 1, 6, 6));
        controls.add(startBtn);
        controls.add(stopBtn);
        controls.add(addBtn);
        controls.add(removeBtn);
        controls.add(intervalLabel);
        controls.add(intervalSlider);

        right.add(tableScroll, BorderLayout.CENTER);
        right.add(controls, BorderLayout.SOUTH);

        getContentPane().setLayout(new BorderLayout(8, 8));
        getContentPane().add(mapPanel, BorderLayout.CENTER);
        getContentPane().add(right, BorderLayout.EAST);

        // Add a few vehicles to start with
        for (int i = 0; i < 6; i++) {
            addInitialVehicle();
        }
        refreshTable();

        // Stop scheduler on close
        addWindowListener(new WindowAdapter() {
            @Override
            public void windowClosing(WindowEvent e) {
                stopSimulation();
                scheduler.shutdownNow();
            }
        });
    }

    private void addInitialVehicle() {
        String id = "V" + (vehicles.size() + 1);
        double lat = minLat + Math.random() * (maxLat - minLat);
        double lon = minLon + Math.random() * (maxLon - minLon);
        vehicles.add(new Vehicle(id, lat, lon));
    }

    private void refreshTable() {
        SwingUtilities.invokeLater(() -> {
            tableModel.setRowCount(0);
            synchronized (vehicles) {
                for (Vehicle v : vehicles) {
                    tableModel.addRow(new Object[]{v.id, String.format("%.6f", v.lat), String.format("%.6f", v.lon), String.format("%.2f", v.speed), String.format("%.1f", v.heading)});
                }
            }
        });
    }

    private void startSimulation(int intervalMs) {
        stopSimulation();
        scheduledTask = scheduler.scheduleAtFixedRate(() -> {
            long now = System.currentTimeMillis();
            synchronized (vehicles) {
                for (Vehicle v : vehicles) {
                    updateVehiclePosition(v, (intervalMs) / 1000.0);
                    v.lastUpdated = now;
                }
            }
            refreshTable();
            mapPanel.repaint();
        }, 0, intervalMs, TimeUnit.MILLISECONDS);
    }

    private void stopSimulation() {
        if (scheduledTask != null && !scheduledTask.isCancelled()) {
            scheduledTask.cancel(true);
            scheduledTask = null;
        }
    }

    // Move vehicle according to speed (m/s) and heading over deltaSeconds. Uses simple equirectangular approx.
    private void updateVehiclePosition(Vehicle v, double deltaSeconds) {
        double distanceMeters = v.speed * deltaSeconds; // meters
        // Earth approx: 1 deg lat ~ 111320 m
        double deltaLat = (distanceMeters * Math.cos(Math.toRadians(v.heading))) / 111320.0;
        double deltaLon = (distanceMeters * Math.sin(Math.toRadians(v.heading))) / (111320.0 * Math.cos(Math.toRadians(v.lat)));
        v.lat += deltaLat;
        v.lon += deltaLon;

        // Random small heading change
        v.heading += (Math.random() - 0.5) * 20.0;
        if (v.heading < 0) v.heading += 360;
        if (v.heading >= 360) v.heading -= 360;

        // Keep inside bounding box, reflect heading
        if (v.lat < minLat) { v.lat = minLat; v.heading = 360 - v.heading; }
        if (v.lat > maxLat) { v.lat = maxLat; v.heading = 360 - v.heading; }
        if (v.lon < minLon) { v.lon = minLon; v.heading = 180 - v.heading; }
        if (v.lon > maxLon) { v.lon = maxLon; v.heading = 180 - v.heading; }

        // Slight random speed variation
        v.speed = Math.max(0.5, v.speed + (Math.random() - 0.5) * 1.0);
    }
}

// Simple map panel to draw vehicles
class MapPanel extends JPanel {
    private final java.util.List<Vehicle> vehicles;
    private final double minLat, maxLat, minLon, maxLon;

    MapPanel(java.util.List<Vehicle> vehicles, double minLat, double maxLat, double minLon, double maxLon) {
        this.vehicles = vehicles;
        this.minLat = minLat; this.maxLat = maxLat; this.minLon = minLon; this.maxLon = maxLon;
        setPreferredSize(new Dimension(600, 600));
        setBackground(Color.WHITE);
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        Graphics2D g2 = (Graphics2D) g.create();
        int w = getWidth();
        int h = getHeight();

        // Draw grid/axes
        g2.setColor(Color.LIGHT_GRAY);
        for (int i = 0; i <= 10; i++) {
            int x = i * w / 10;
            int y = i * h / 10;
            g2.drawLine(x, 0, x, h);
            g2.drawLine(0, y, w, y);
        }

        // Labels for bounding box
        g2.setColor(Color.DARK_GRAY);
        g2.drawString(String.format("Lat: %.6f to %.6f", minLat, maxLat), 10, 15);
        g2.drawString(String.format("Lon: %.6f to %.6f", minLon, maxLon), 10, 30);

        // Draw each vehicle as a colored dot with ID
        synchronized (vehicles) {
            int idx = 0;
            for (Vehicle v : vehicles) {
                Point p = toPoint(v.lat, v.lon, w, h);
                Color color = getColorForIndex(idx++);
                g2.setColor(color);
                int r = 8;
                g2.fillOval(p.x - r/2, p.y - r/2, r, r);
                g2.setColor(Color.BLACK);
                g2.drawString(v.id + String.format(" (%.1f m/s)", v.speed), p.x + 6, p.y - 6);
                // heading arrow
                double hx = p.x + Math.cos(Math.toRadians(v.heading)) * 18;
                double hy = p.y - Math.sin(Math.toRadians(v.heading)) * 18;
                g2.drawLine(p.x, p.y, (int)hx, (int)hy);
            }
        }

        g2.dispose();
    }

    private Color getColorForIndex(int i) {
        Color[] palette = new Color[]{Color.RED, Color.BLUE, Color.GREEN.darker(), Color.MAGENTA, Color.ORANGE, Color.CYAN.darker(), Color.PINK, Color.YELLOW.darker()};
        return palette[i % palette.length];
    }

    private Point toPoint(double lat, double lon, int w, int h) {
        double xRatio = (lon - minLon) / (maxLon - minLon);
        double yRatio = (maxLat - lat) / (maxLat - minLat); // lat decreases downwards
        int x = (int) Math.round(xRatio * (w - 20)) + 10;
        int y = (int) Math.round(yRatio * (h - 20)) + 10;
        return new Point(x, y);
    }
}
