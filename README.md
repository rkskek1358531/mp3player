# mp3player
java maven mp3player

## 코드
### Main 자바파일
```java
package org.example;

import javafx.application.Application;
import javafx.application.Platform;
import javafx.beans.property.*;
import javafx.geometry.Insets;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.*;
import javafx.scene.input.KeyCode;
import javafx.scene.layout.*;
import javafx.scene.media.Media;
import javafx.scene.media.MediaPlayer;
import javafx.scene.media.MediaPlayer.Status;
import javafx.stage.DirectoryChooser;
import javafx.stage.Stage;
import javafx.util.Duration;

import java.io.File;
import java.net.URL;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import org.jaudiotagger.audio.AudioFile;
import org.jaudiotagger.audio.AudioFileIO;
import org.jaudiotagger.tag.FieldKey;
import org.jaudiotagger.tag.Tag;

public class Main extends Application {

    enum RepeatMode {
        NONE, REPEAT_ALL, REPEAT_ONE;
        public RepeatMode next() {
            switch (this) {
                case NONE: return REPEAT_ALL;
                case REPEAT_ALL: return REPEAT_ONE;
                default: return NONE;
            }
        }
        public String label() {
            switch (this) {
                case NONE: return "일회 재생";
                case REPEAT_ALL: return "전체 반복";
                case REPEAT_ONE: return "한 곡 반복";
                default: return "";
            }
        }
    }

    static class LyricLine {
        final double timeSec;
        final String text;
        Label label;
        LyricLine(double timeSec, String text) { this.timeSec = timeSec; this.text = text; }
    }

    private final javafx.collections.ObservableList<Path> playlist = javafx.collections.FXCollections.observableArrayList();
    private final ListView<Path> listView = new ListView<>(playlist);

    private final IntegerProperty currentIndex = new SimpleIntegerProperty(-1);
    private final ObjectProperty<MediaPlayer> player = new SimpleObjectProperty<>(null);
    private final ObjectProperty<RepeatMode> repeatMode = new SimpleObjectProperty<>(RepeatMode.NONE);

    private final Slider seekSlider = new Slider(0, 100, 0);
    private final Slider volumeSlider = new Slider(0, 1, 0.8);
    private final Label timeLabel = new Label("00:00 / 00:00");
    private final Label titleLabel = new Label("폴더를 선택해 주세요");
    private final Button playPauseBtn = new Button("재생");
    private final Button prevBtn = new Button("이전");
    private final Button nextBtn = new Button("다음");
    private final Button repeatBtn = new Button();

    private final ScrollPane lyricsPane = new ScrollPane();
    private final VBox lyricsBox = new VBox(4);
    private final BooleanProperty lyricsSynced = new SimpleBooleanProperty(false);
    private List<LyricLine> linesForTrack = Collections.emptyList();
    private int currentLyricIndex = -1;

    @Override
    public void start(Stage stage) {
        stage.setTitle("JavaFX MP3 Player");

        Button openFolderBtn = new Button("폴더 선택");
        openFolderBtn.setOnAction(e -> chooseFolder(stage));
        HBox topBar = new HBox(10, openFolderBtn, titleLabel);
        topBar.setAlignment(Pos.CENTER_LEFT);
        topBar.setPadding(new Insets(10));
        topBar.getStyleClass().add("topbar");

        listView.setCellFactory(lv -> new ListCell<>() {
            @Override
            protected void updateItem(Path item, boolean empty) {
                super.updateItem(item, empty);
                setText(empty || item == null ? null : item.getFileName().toString());
            }
        });
        listView.setOnMouseClicked(e -> {
            if (e.getClickCount() == 2) {
                int idx = listView.getSelectionModel().getSelectedIndex();
                if (idx >= 0) playAt(idx);
            }
        });

        lyricsPane.setFitToWidth(true);
        lyricsPane.setContent(lyricsBox);
        lyricsPane.setPadding(new Insets(10));

        SplitPane split = new SplitPane();
        split.setDividerPositions(0.4);
        split.getItems().addAll(listView, lyricsPane);

        playPauseBtn.setOnAction(e -> togglePlayPause());
        prevBtn.setOnAction(e -> playPrevious());
        nextBtn.setOnAction(e -> playNext());
        updateRepeatButtonText();
        repeatBtn.setOnAction(e -> {
            repeatMode.set(repeatMode.get().next());
            updateRepeatButtonText();
        });

        seekSlider.setDisable(true);
        seekSlider.setOnMousePressed(e -> scrubToSlider());
        seekSlider.setOnMouseReleased(e -> scrubToSlider());

        volumeSlider.valueProperty().addListener((obs, ov, nv) -> {
            if (player.get() != null) player.get().setVolume(nv.doubleValue());
        });

        HBox transport = new HBox(10, prevBtn, playPauseBtn, nextBtn, repeatBtn);
        transport.setAlignment(Pos.CENTER_LEFT);

        HBox timeRow = new HBox(10, new Label("진행"), seekSlider, timeLabel);
        timeRow.setAlignment(Pos.CENTER_LEFT);
        HBox volRow = new HBox(10, new Label("볼륨"), volumeSlider);
        volRow.setAlignment(Pos.CENTER_LEFT);

        VBox bottom = new VBox(8, transport, timeRow, volRow);
        bottom.setPadding(new Insets(10));
        bottom.getStyleClass().add("bottombar");

        BorderPane root = new BorderPane();
        root.setTop(topBar);
        root.setCenter(split);
        root.setBottom(bottom);

        Scene scene = new Scene(root, 1000, 620);

        // CSS 로딩: 클래스패스 → 컨텍스트CL → 개발 경로 → 임시 생성 폴백
        URL cssUrl = null;
        cssUrl = (cssUrl == null) ? Main.class.getResource("/styles.css") : cssUrl;                 // 클래스패스 루트
        cssUrl = (cssUrl == null) ? Thread.currentThread().getContextClassLoader().getResource("styles.css") : cssUrl; // 컨텍스트 CL
        cssUrl = (cssUrl == null) ? Main.class.getResource("styles.css") : cssUrl;                  // org/example/style.css

        if (cssUrl != null) {
            System.out.println("[CSS] Loaded from: " + cssUrl);
            scene.getStylesheets().add(cssUrl.toExternalForm());
        } else {
            Path devCss = Paths.get("src/main/resources/styles.css");
            if (Files.exists(devCss)) {
                String uri = devCss.toUri().toString();
                System.out.println("[CSS] Fallback file path: " + uri);
                scene.getStylesheets().add(uri);
            } else {
                String css = """
                    /* dark theme minimal fallback */
                    .root { -fx-background-color:#111418; -fx-text-fill:#e8eaed; -fx-font-family:"Noto Sans CJK KR","Malgun Gothic","Segoe UI",sans-serif; -fx-font-size:14px; }
                    .topbar, .bottombar { -fx-background-color:#1b1f24; -fx-border-color:#2a2f36; }
                    .topbar { -fx-border-width:0 0 1 0; } .bottombar { -fx-border-width:1 0 0 0; }
                    .button { -fx-background-radius:12; -fx-background-color:#2a2f36; -fx-text-fill:#e8eaed; -fx-padding:8 14 8 14; -fx-font-size:13px; }
                    .button:hover { -fx-background-color:#354050; }
                    .list-view { -fx-background-color:#111418; -fx-control-inner-background:#111418; -fx-border-color:#2a2f36; }
                    .list-cell { -fx-text-fill:#e8eaed; -fx-padding:8 12 8 12; }
                    .list-cell:filled:selected, .list-cell:filled:selected:hover { -fx-background-color:#2f6feb; -fx-text-fill:white; }
                    .label { -fx-text-fill:#e8eaed; }
                    .lyric-line { -fx-font-size:15px; -fx-line-spacing:4px; -fx-text-fill:#cfd3da; }
                    .lyric-current { -fx-font-size:16px; -fx-font-weight:bold; -fx-text-fill:#ffffff; }
                    .scroll-pane { -fx-background:#111418; -fx-control-inner-background:#111418; }
                    .scroll-pane .viewport { -fx-background-color:#111418; }
                    .split-pane { -fx-background-color:#111418; } .split-pane-divider { -fx-background-color:#2a2f36; }
                    .slider .track { -fx-background-color:#2a2f36; } .slider .thumb { -fx-background-color:#2f6feb; -fx-background-radius:50%; }
                    .dialog-pane { -fx-background-color:#1b1f24; }
                    .dialog-pane .header-panel { -fx-background-color:#2a2f36; -fx-text-fill:#e8eaed; }
                    .dialog-pane .content.label { -fx-text-fill:#e8eaed; }
                    .dialog-pane .button-bar .button { -fx-background-color:#2a2f36; -fx-text-fill:#e8eaed; }
                    .dialog-pane .button-bar .button:hover { -fx-background-color:#354050; }
                """;
                try {
                    Path tmp = Files.createTempFile("style-fallback-", ".css");
                    Files.writeString(tmp, css);
                    String uri = tmp.toUri().toString();
                    System.out.println("[CSS] Generated temp stylesheet: " + uri);
                    scene.getStylesheets().add(uri);
                } catch (Exception ex) {
                    throw new IllegalStateException("styles.css를 찾거나 생성할 수 없음: " + ex.getMessage(), ex);
                }
            }
        }

        stage.setScene(scene);
        stage.show();

        scene.setOnKeyPressed(e -> {
            if (e.getCode() == KeyCode.SPACE) togglePlayPause();
            else if (e.getCode() == KeyCode.RIGHT) playNext();
            else if (e.getCode() == KeyCode.LEFT) playPrevious();
            else if (e.getCode() == KeyCode.R) {
                repeatMode.set(repeatMode.get().next());
                updateRepeatButtonText();
            }
        });

        player.addListener((obs, oldP, newP) -> {
            if (oldP != null) {
                oldP.currentTimeProperty().removeListener(seekListener);
                oldP.statusProperty().removeListener(statusListener);
                oldP.setOnEndOfMedia(null);
                oldP.dispose();
            }
            if (newP != null) {
                newP.currentTimeProperty().addListener(seekListener);
                newP.statusProperty().addListener(statusListener);
                newP.setOnEndOfMedia(this::onEndOfMedia);
                newP.setVolume(volumeSlider.getValue());
                seekSlider.setDisable(false);
            } else {
                seekSlider.setDisable(true);
            }
        });
    }

    private void chooseFolder(Stage stage) {
        DirectoryChooser chooser = new DirectoryChooser();
        chooser.setTitle("MP3 폴더 선택");
        File dir = chooser.showDialog(stage);
        if (dir == null) return;

        List<Path> files;
        try (Stream<Path> s = Files.walk(dir.toPath())) {
            files = s.filter(p -> !Files.isDirectory(p))
                    .filter(p -> p.getFileName().toString().toLowerCase().endsWith(".mp3"))
                    .sorted()
                    .collect(Collectors.toList());
        } catch (Exception ex) {
            showError("폴더를 읽는 중 오류: " + ex.getMessage());
            return;
        }

        playlist.setAll(files);
        if (playlist.isEmpty()) {
            titleLabel.setText("선택한 폴더에 mp3 없음");
            currentIndex.set(-1);
            stopAndClear();
            return;
        }
        titleLabel.setText("트랙 " + playlist.size() + "개");
        listView.getSelectionModel().select(0);
        playAt(0);
    }

    private void playAt(int index) {
        if (index < 0 || index >= playlist.size()) return;
        currentIndex.set(index);
        listView.getSelectionModel().select(index);
        listView.scrollTo(index);

        Path track = playlist.get(index);
        loadLyrics(track);

        try {
            Media media = new Media(track.toUri().toString());
            MediaPlayer mp = new MediaPlayer(media);
            player.set(mp);

            titleLabel.setText(track.getFileName().toString());
            mp.setOnReady(() -> {
                Duration total = mp.getTotalDuration();
                updateTimeLabels(Duration.ZERO, total);
                bindSeekSlider(mp);
                mp.play();
                playPauseBtn.setText("일시정지");
            });
        } catch (Exception ex) {
            showError("재생 불가: " + track.getFileName());
            playNext();
        }
    }

    private void togglePlayPause() {
        MediaPlayer mp = player.get();
        if (mp == null) return;
        if (mp.getStatus() == Status.PLAYING) {
            mp.pause();
            playPauseBtn.setText("재생");
        } else {
            mp.play();
            playPauseBtn.setText("일시정지");
        }
    }

    private void playNext() {
        if (playlist.isEmpty()) return;
        int idx = currentIndex.get();
        if (idx < playlist.size() - 1) playAt(idx + 1);
        else if (repeatMode.get() == RepeatMode.REPEAT_ALL) playAt(0);
        else stopPlaybackKeepPosition();
    }

    private void playPrevious() {
        if (playlist.isEmpty()) return;
        int idx = currentIndex.get();
        if (idx > 0) playAt(idx - 1);
        else if (repeatMode.get() == RepeatMode.REPEAT_ALL) playAt(playlist.size() - 1);
        else stopPlaybackKeepPosition();
    }

    private void onEndOfMedia() {
        if (repeatMode.get() == RepeatMode.REPEAT_ONE) {
            MediaPlayer mp = player.get();
            if (mp != null) {
                mp.seek(Duration.ZERO);
                mp.play();
            }
        } else playNext();
    }

    private void stopAndClear() {
        MediaPlayer mp = player.get();
        if (mp != null) {
            mp.stop();
            player.set(null);
        }
        seekSlider.setValue(0);
        timeLabel.setText("00:00 / 00:00");
        playPauseBtn.setText("재생");
        clearLyrics();
    }

    private void stopPlaybackKeepPosition() {
        MediaPlayer mp = player.get();
        if (mp != null) {
            mp.stop();
            playPauseBtn.setText("재생");
        }
    }

    private void updateRepeatButtonText() {
        repeatBtn.setText(repeatMode.get().label());
    }

    private void bindSeekSlider(MediaPlayer mp) {
        seekSlider.setMin(0);
        seekSlider.setMax(mp.getTotalDuration().toSeconds());
        seekSlider.valueChangingProperty().addListener((obs, was, isChanging) -> {
            if (!isChanging && player.get() != null) {
                player.get().seek(Duration.seconds(seekSlider.getValue()));
            }
        });
    }

    private void scrubToSlider() {
        MediaPlayer mp = player.get();
        if (mp != null) mp.seek(Duration.seconds(seekSlider.getValue()));
    }

    private final javafx.beans.value.ChangeListener<Duration> seekListener = (obs, oldV, newV) -> {
        MediaPlayer mp = player.get();
        if (mp == null) return;
        Duration total = mp.getTotalDuration();
        if (total == null || total.lessThanOrEqualTo(Duration.ZERO)) return;
        if (!seekSlider.isValueChanging()) seekSlider.setValue(newV.toSeconds());
        updateTimeLabels(newV, total);
        updateLyricsHighlight(newV.toSeconds());
    };

    private final javafx.beans.value.ChangeListener<Status> statusListener = (obs, oldS, newS) -> {
        if (newS == Status.PLAYING) playPauseBtn.setText("일시정지");
        else playPauseBtn.setText("재생");
    };

    private void updateTimeLabels(Duration current, Duration total) {
        timeLabel.setText(formatTime(current) + " / " + formatTime(total));
    }

    private String formatTime(Duration d) {
        int seconds = (int)Math.floor(d.toSeconds());
        return String.format("%02d:%02d", seconds/60, seconds%60);
    }

    private void showError(String msg) {
        Alert alert = new Alert(Alert.AlertType.ERROR, msg, ButtonType.OK);
        alert.setHeaderText("오류");
        alert.showAndWait();
    }

    private void clearLyrics() {
        linesForTrack = Collections.emptyList();
        currentLyricIndex = -1;
        lyricsBox.getChildren().clear();
    }

    private void loadLyrics(Path track) {
        clearLyrics();
        Path lrc = replaceExt(track, ".lrc");
        List<LyricLine> lines = null;
        if (Files.exists(lrc)) {
            lines = parseLrc(lrc);
            lyricsSynced.set(true);
        } else {
            String text = readEmbeddedLyrics(track);
            if (text != null && !text.isBlank()) {
                lines = text.lines().map(s -> new LyricLine(-1, s)).toList();
                lyricsSynced.set(false);
            }
        }
        if (lines == null || lines.isEmpty()) {
            lines = List.of(new LyricLine(-1, "가사가 없습니다"));
            lyricsSynced.set(false);
        }

        for (LyricLine line : lines) {
            Label lab = new Label(line.text);
            lab.setWrapText(true);
            lab.getStyleClass().add("lyric-line");
            line.label = lab;
            lyricsBox.getChildren().add(lab);
        }
        linesForTrack = lines;
        currentLyricIndex = -1;
    }

    private static Path replaceExt(Path p, String newExt) {
        String name = p.getFileName().toString();
        int dot = name.lastIndexOf('.');
        String base = dot > 0 ? name.substring(0, dot) : name;
        return p.getParent().resolve(base + newExt);
    }

    private List<LyricLine> parseLrc(Path lrcPath) {
        try {
            List<String> all = Files.readAllLines(lrcPath);
            List<LyricLine> out = new ArrayList<>();
            for (String raw : all) {
                String line = raw.trim();
                if (line.isEmpty()) continue;
                if (line.matches("^\\[(ti|ar|al|by|offset):.*\\]$")) continue;
                java.util.regex.Matcher m = java.util.regex.Pattern
                        .compile("\\[(\\d{1,2}):(\\d{2})(?:\\.(\\d{1,2}))?\\]")
                        .matcher(line);
                double firstTs = -1;
                int last = 0;
                while (m.find()) {
                    int mm = Integer.parseInt(m.group(1));
                    int ss = Integer.parseInt(m.group(2));
                    int cs = m.group(3) != null ? Integer.parseInt(m.group(3)) : 0;
                    double frac = (m.group(3) == null) ? 0.0 : cs / Math.pow(10, m.group(3).length());
                    double t = mm*60 + ss + frac;
                    if (firstTs < 0) firstTs = t;
                    last = m.end();
                }
                String text = line.substring(last).trim();
                if (firstTs >= 0 && !text.isEmpty()) out.add(new LyricLine(firstTs, text));
                else if (!text.isEmpty()) out.add(new LyricLine(-1, text));
            }
            out.sort(Comparator.comparingDouble(ll -> ll.timeSec < 0 ? Double.MAX_VALUE : ll.timeSec));
            return out;
        } catch (Exception e) {
            showError("LRC 파싱 실패: " + e.getMessage());
            return List.of();
        }
    }

    private String readEmbeddedLyrics(Path track) {
        try {
            AudioFile af = AudioFileIO.read(track.toFile());
            Tag tag = af.getTag();
            if (tag == null) return null;
            return tag.getFirst(FieldKey.LYRICS);
        } catch (Exception e) {
            return null;
        }
    }

    private void updateLyricsHighlight(double curSec) {
        if (!lyricsSynced.get() || linesForTrack.isEmpty()) return;
        int idx = -1;
        for (int i = 0; i < linesForTrack.size(); i++) {
            double t = linesForTrack.get(i).timeSec;
            if (t < 0) continue;
            if (t <= curSec) idx = i; else break;
        }
        if (idx != -1 && idx != currentLyricIndex) {
            if (currentLyricIndex >= 0 && currentLyricIndex < linesForTrack.size()) {
                linesForTrack.get(currentLyricIndex).label.getStyleClass().remove("lyric-current");
            }
            currentLyricIndex = idx;
            LyricLine cur = linesForTrack.get(idx);
            cur.label.getStyleClass().add("lyric-current");

            double contentHeight = lyricsBox.getHeight();
            double y = cur.label.getBoundsInParent().getMinY();
            double v = (y / Math.max(1, contentHeight));
            lyricsPane.setVvalue(Math.min(1.0, Math.max(0.0, v)));
        }
    }

    @Override
    public void stop() {
        MediaPlayer mp = player.get();
        if (mp != null) mp.dispose();
        Platform.exit();
    }

    public static void main(String[] args) {
        launch(args);
    }
}


```

### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>dev.play</groupId>
    <artifactId>mp3player</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <javafx.version>21.0.3</javafx.version>
    </properties>

    <dependencies>
        <!-- JavaFX -->
        <dependency>
            <groupId>org.openjfx</groupId>
            <artifactId>javafx-controls</artifactId>
            <version>${javafx.version}</version>
        </dependency>
        <dependency>
            <groupId>org.openjfx</groupId>
            <artifactId>javafx-media</artifactId>
            <version>${javafx.version}</version>
        </dependency>

        <!-- 최신 jaudiotagger -->
        <dependency>
            <groupId>net.jthink</groupId>
            <artifactId>jaudiotagger</artifactId>
            <version>3.0.1</version>
        </dependency>
    </dependencies>

    <build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <release>${maven.compiler.release}</release>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.openjfx</groupId>
                <artifactId>javafx-maven-plugin</artifactId>
                <version>0.0.8</version>
                <configuration>
                    <mainClass>org.example.Main</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>


```

### styles.css(적용 되는지를 모르겠음. 하지만 혹시 모르니까 올림)
```css
/* ===== 전체 ===== */
.root {
    -fx-font-family: "Noto Sans CJK KR", "Malgun Gothic", "Segoe UI", sans-serif;
    -fx-font-size: 14px;
    -fx-background-color: #111418; /* 메인 배경 */
    -fx-text-fill: #e8eaed;
}

/* ===== 상단/하단 바 ===== */
.topbar, .bottombar {
    -fx-background-color: #1b1f24;
    -fx-border-color: #2a2f36;
}
.topbar {
    -fx-border-width: 0 0 1 0;
}
.bottombar {
    -fx-border-width: 1 0 0 0;
}

/* ===== 버튼 ===== */
.button {
    -fx-background-radius: 12;
    -fx-background-color: #2a2f36;
    -fx-text-fill: #e8eaed;
    -fx-padding: 8 14 8 14;
    -fx-font-size: 13px;
}
.button:hover {
    -fx-background-color: #354050;
}

/* ===== 리스트뷰 ===== */
.list-view {
    -fx-background-color: #111418;
    -fx-control-inner-background: #111418;
    -fx-border-color: #2a2f36;
}
.list-cell {
    -fx-text-fill: #e8eaed;
    -fx-padding: 8 12 8 12;
}
.list-cell:filled:selected, .list-cell:filled:selected:hover {
    -fx-background-color: #2f6feb;
    -fx-text-fill: white;
}

/* ===== 라벨 ===== */
.label {
    -fx-text-fill: #e8eaed;
}

/* ===== 타이틀 ===== */
.title {
    -fx-font-size: 18px;
    -fx-font-weight: 700;
    -fx-text-fill: #ffffff;
}

/* ===== 가사 영역 ===== */
.lyric-line {
    -fx-font-size: 15px;
    -fx-line-spacing: 4px;
    -fx-text-fill: #cfd3da;
}
.lyric-current {
    -fx-font-size: 16px;
    -fx-font-weight: bold;
    -fx-text-fill: #ffffff;
}

/* ===== 슬라이더 ===== */
.slider {
    -fx-control-inner-background: #2a2f36;
}
.slider .track {
    -fx-background-color: #2a2f36;
}
.slider .thumb {
    -fx-background-color: #2f6feb;
    -fx-background-radius: 50%;
}

/* ===== 스크롤팬 ===== */
.scroll-pane {
    -fx-background: #111418;
    -fx-control-inner-background: #111418;
}
.scroll-pane .viewport {
    -fx-background-color: #111418;
}

/* ===== 스플릿팬 ===== */
.split-pane {
    -fx-background-color: #111418;
}
.split-pane-divider {
    -fx-background-color: #2a2f36;
}

/* ===== 알림창(AlertDialog) ===== */
.dialog-pane {
    -fx-background-color: #1b1f24;
}
.dialog-pane .header-panel {
    -fx-background-color: #2a2f36;
    -fx-text-fill: #e8eaed;
}
.dialog-pane .content.label {
    -fx-text-fill: #e8eaed;
}
.dialog-pane .button-bar .button {
    -fx-background-color: #2a2f36;
    -fx-text-fill: #e8eaed;
}
.dialog-pane .button-bar .button:hover {
    -fx-background-color: #354050;
}

```

## 실행 결과 화면
![](https://velog.velcdn.com/images/dovmf3910/post/43bda536-b863-482f-9048-4c214e60868e/image.png)

![](https://velog.velcdn.com/images/dovmf3910/post/355af67b-fcb6-486a-a8bf-c7c0216f4598/image.png)

![](https://velog.velcdn.com/images/dovmf3910/post/5ace1cc6-c0a9-4c32-8862-22c04481f074/image.png)
